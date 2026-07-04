# StreamableHTTP Reference

Technical reference for `SKILL.md`. Read this when applying Step 3 (decision matrix) and Step 6 (scaffolding) of that skill's process.

## 1. What each flag actually changes

**`stateless_http`** (Python SDK) / `sessionIdGenerator: undefined` (TypeScript SDK)
Controls whether the server keeps a persistent session tied to a session ID across requests.

- `False` / a real session ID generator (default): the server issues a session ID on initialization and keeps state tied to it across requests. This is required for anything the server needs to send back to the client asynchronously.
- `True` / `undefined`: every request is handled in isolation, with no session ID and no shared state between requests. This is what makes StreamableHTTP safe to run behind a load balancer or across multiple replicas — any instance can handle any request, because none of them are holding session state the others don't have.

**Consequence of stateless mode:** because there is no persistent session or open channel back to a specific client, anything that depends on the server reaching back to the client mid-session stops working reliably — sampling requests, progress/log notifications, and resource-subscription updates. This is not a separate setting; it is a direct consequence of removing the session. If a tool needs any of these, stateless mode is the wrong choice regardless of `json_response`.

**`json_response`** (Python) / `enableJsonResponse` (TypeScript)
Controls the response format for a single request/response cycle.

- `False` (default): the server responds using a Server-Sent Events (SSE) stream, which allows multiple messages (including notifications) to be sent for one request.
- `True`: the server returns a single plain JSON body instead of opening an SSE stream. Simpler for infrastructure that doesn't handle long-lived SSE well (API Gateway, Lambda, some load balancers), but it means only the final result comes back — per the TypeScript SDK's own tracked behavior, notifications tied to that request (e.g., progress notifications) are not delivered when this is on.

## 2. Decision matrix

Apply in this order. Server-push need is checked first because it can override everything else.

| Server-push need (sampling / progress / subscriptions)? | Deployment shape | Recommendation |
|---|---|---|
| Yes | Any | **Stop.** Stateless StreamableHTTP cannot reliably support this. Recommend stateful StreamableHTTP (single instance, real session IDs, `json_response=False`) or stdio if network access isn't actually required. Do not scaffold stateless mode. |
| No | Single local/dev instance | `stateless_http=False` (or omit), `json_response=False`. Stateful is fine and simplest here; there's no scaling reason to give up the session. |
| No | Hosted behind a load balancer, multiple replicas, or serverless/edge | `stateless_http=True`, `json_response=True`. No shared session store is needed across instances, and JSON responses avoid SSE issues with infrastructure that doesn't hold long-lived connections well. |
| No | Single hosted instance, no load balancer, but network-reachable | Either works. Prefer `stateless_http=True, json_response=True` for simplicity, unless the user wants session continuity across calls for some other reason — ask if unclear. |

State whichever row applied, and why, in plain language, before scaffolding.

## 3. Python (MCP Python SDK — `mcp.server.fastmcp`)

### Stateful (default) — single instance, no push requirement violated by stateless mode isn't a concern here since this keeps the session

```python
from mcp.server.fastmcp import FastMCP

mcp = FastMCP("{{server_name}}")

@mcp.tool()
async def {{tool_name}}({{args}}) -> str:
    """{{One-line description tied to the connection target, e.g. 'Fetch an issue from GitHub by number.'}}"""
    # Call {{connection_target}} here.
    ...

if __name__ == "__main__":
    mcp.run(transport="streamable-http")
```

### Stateless + JSON response — hosted behind a load balancer / serverless

```python
from mcp.server.fastmcp import FastMCP

mcp = FastMCP("{{server_name}}", stateless_http=True, json_response=True)

@mcp.tool()
async def {{tool_name}}({{args}}) -> str:
    """{{One-line description tied to the connection target.}}"""
    # Call {{connection_target}} here.
    ...

if __name__ == "__main__":
    mcp.run(transport="streamable-http")
```

## 4. TypeScript (`@modelcontextprotocol/sdk`)

### Stateful (default) — single instance

```typescript
import express from "express";
import { randomUUID } from "node:crypto";
import { McpServer } from "@modelcontextprotocol/sdk/server/mcp.js";
import { StreamableHTTPServerTransport } from "@modelcontextprotocol/sdk/server/streamableHttp.js";

const server = new McpServer({ name: "{{server_name}}", version: "1.0.0" });

server.tool(
  "{{tool_name}}",
  "{{One-line description tied to the connection target.}}",
  { /* {{input schema}} */ },
  async ({ /* args */ }) => {
    // Call {{connection_target}} here.
    return { content: [{ type: "text", text: "..." }] };
  }
);

const app = express();
app.use(express.json());
const transports: Record<string, StreamableHTTPServerTransport> = {};

app.post("/mcp", async (req, res) => {
  const sessionId = req.headers["mcp-session-id"] as string | undefined;
  const transport =
    sessionId && transports[sessionId]
      ? transports[sessionId]
      : new StreamableHTTPServerTransport({
          sessionIdGenerator: () => randomUUID(),
          onsessioninitialized: (id) => { transports[id] = transport; }
        });
  await server.connect(transport);
  await transport.handleRequest(req, res, req.body);
});

app.listen(3000);
```

### Stateless + JSON response — hosted behind a load balancer / serverless

```typescript
import express from "express";
import { McpServer } from "@modelcontextprotocol/sdk/server/mcp.js";
import { StreamableHTTPServerTransport } from "@modelcontextprotocol/sdk/server/streamableHttp.js";

const app = express();
app.use(express.json());

app.post("/mcp", async (req, res) => {
  // New server + transport per request: keeps requests fully isolated,
  // which is required in stateless mode when multiple clients connect concurrently.
  const server = new McpServer({ name: "{{server_name}}", version: "1.0.0" });

  server.tool(
    "{{tool_name}}",
    "{{One-line description tied to the connection target.}}",
    { /* {{input schema}} */ },
    async ({ /* args */ }) => {
      // Call {{connection_target}} here.
      return { content: [{ type: "text", text: "..." }] };
    }
  );

  const transport = new StreamableHTTPServerTransport({
    sessionIdGenerator: undefined,
    enableJsonResponse: true
  });

  res.on("close", () => {
    transport.close();
    server.close();
  });

  await server.connect(transport);
  await transport.handleRequest(req, res, req.body);
});

app.listen(3000);
```

## 5. Placeholders to fill from the four required inputs

- `{{server_name}}` — short name for the server, derived from the connection target.
- `{{tool_name}}` / `{{args}}` / input schema — derived from what the connection target needs to do (e.g., a GitHub target likely needs `owner`, `repo`, `issue_number`).
- `{{connection_target}}` — the actual API/service call (GitHub REST/GraphQL API, an email provider's API, etc.). This skill does not include real credentials or auth code — flag that as a separate, explicit request if the user needs it.
