---
name: streamable-http-mcp-builder
description: Use when an MCP server needs to run over the StreamableHTTP transport instead of stdio — a rare, single-purpose scenario distinct from local stdio servers. Interviews the user on connection target, SDK/language, deployment shape, and server-push needs, recommends stateless_http and json_response settings with stated reasoning, and scaffolds the server from the paired reference file. Do not use for stdio servers, tool logic, testing, deployment, or optimization work.
---

# StreamableHTTP MCP Builder

## Overview

Most MCP servers built in this project run over stdio, and that has its own skill. StreamableHTTP only comes up rarely — when a server needs to be reachable over a network instead of launched as a local subprocess. Because it is rare, there is no built-up instinct for catching a bad configuration, and the two flags that matter — `stateless_http` and `json_response` — are easy to set on autopilot without checking whether they actually fit the deployment.

Normal prompting fails here in a specific way: an AI asked to "build a StreamableHTTP MCP server" will often just copy the most common example it has seen (frequently `stateless_http=True, json_response=True`) without checking whether the deployment shape calls for it, or whether the tools involved need server-initiated messages that those settings quietly disable.

This skill exists to force that check every time: gather the four facts that actually determine the right configuration, recommend the settings with plain-language reasoning tied to those facts, and only then scaffold the server. It does not test, deploy, or optimize the result — that loop belongs to the user.

## When to Use

Use this skill when:

- A user or task needs an MCP server reachable over HTTP/network, not just launched locally via stdio.
- The user mentions deploying an MCP server remotely, behind a load balancer, on serverless/edge infrastructure, or otherwise accessible outside the local machine.
- The user explicitly says "StreamableHTTP," "remote MCP server," "stateless," "no session ID," "HTTP transport," or similar.
- An MCP server is being created or modified and the transport has not yet been fixed to stdio.

## When Not to Use

Do not use this skill when:

- The server only needs to run locally via stdio (use the existing stdio skill instead).
- The tools involved require sampling, progress notifications, or resource-subscription updates as core functionality. StreamableHTTP in stateless mode removes the persistent channel these depend on — flag this to the user and stop instead of forcing the transport. Recommend stateful StreamableHTTP, or stdio, instead.
- The task is unrelated to transport configuration (writing tool logic, unrelated application code, UI work).
- The task is testing, deployment, or post-build optimization of an already-scaffolded server — those stay with the user's own review process, not this skill.
- Connection target, SDK, and deployment shape have already been confirmed earlier in the current project or conversation. Reuse those answers. Do not ask again.

## Required Inputs

Collect or confirm all four before recommending any setting or generating any code. Ask one at a time if missing; do not default silently.

1. **Connection target** — what the server talks to (e.g., GitHub, email, a custom API, a database).
2. **SDK / language** — Python or TypeScript (or state plainly if another official MCP SDK is required and this skill's reference does not cover it).
3. **Deployment shape** — a single local/dev instance, versus hosted behind a load balancer, run across multiple replicas, or deployed to serverless/edge infrastructure. This is the deciding input for `stateless_http`.
4. **Server-push requirement** — does any tool need to send sampling requests, progress/log notifications, or resource-subscription updates back to the client mid-session? This can disqualify stateless StreamableHTTP entirely.

If any input is missing, ask exactly one focused question before continuing.

## Process

### Step 1: Confirm the trigger

Verify the server genuinely needs a network-reachable transport. If the user actually wants a local-only server, stop and redirect to the stdio skill instead of continuing here.

### Step 2: Collect the four required inputs

Ask one at a time, in the order listed above. Skip any input already confirmed earlier in this conversation or project — do not re-ask.

### Step 3: Apply the decision matrix

Use `streamable-http-reference.md` to turn the deployment shape and server-push answers into a `stateless_http` and `json_response` recommendation. Never hardcode either flag — every value must trace to one of the four answers.

If the server-push answer is "yes," say plainly that stateless StreamableHTTP is the wrong fit for at least part of the workload, and offer the two real alternatives: stateful StreamableHTTP (keeps the session and the push channel, but gives up the load-balancing/serverless simplicity of stateless mode), or stdio (if the server doesn't actually need to be network-reachable). Do not scaffold a stateless server that will silently drop the notifications the user needs.

### Step 4: State the recommendation in plain language

The user does not code. State each flag's value and the one-sentence reason tied to their answers — not the underlying protocol mechanics. Example shape:

> Recommended: `stateless_http=True`, `json_response=True`. Reason: you're deploying behind a load balancer with multiple instances, and none of your tools need to push updates back to the client, so a stateless request/response server is the simpler and more reliable fit.

### Step 5: Confirm before generating code

Ask: "Proceed with this configuration?" Do not scaffold anything until the user confirms.

### Step 6: Scaffold

Once confirmed, generate the server file in the confirmed SDK/language using the matching pattern from `streamable-http-reference.md`, with one example tool reflecting the stated connection target.

### Step 7: Stop

Do not write test code, deployment scripts (Docker, load balancer config, CI), or optimization suggestions. Those belong to the user's own test-and-send-back-to-Cursor loop, which happens after this skill's output, not as part of it.

## Output

The final output of this skill must include:

- A plain-language statement of the recommended `stateless_http` and `json_response` values, each with a one-sentence reason tied to the user's actual answers.
- One scaffolded MCP server file in the confirmed SDK/language, correctly configured, with one example tool matching the stated connection target.
- Nothing else. No test harness, no deployment configuration, no optimization pass, no unrelated files.

## Quality Bar

A correct result:

- Traces every flag value to a specific answer (deployment shape, server-push need) — never to a copied default.
- Uses plain language for the recommendation, since the user cannot evaluate protocol jargon.
- Produces a runnable scaffold, not pseudocode or a fragment missing imports.
- Stops at the scaffold. It does not drift into testing, deployment, or optimization territory.

## Safety and Boundaries

This skill must not:

- Set `stateless_http` or `json_response` without stating the reasoning tied to the user's answers.
- Proceed with a stateless scaffold when the server-push answer was "yes." Stop and say so instead.
- Invent SDK behavior not present in `streamable-http-reference.md`.
- Add authentication or secrets handling unless the user explicitly asks for it in a separate request — flag the gap rather than inventing a security pattern.
- Claim a transport can do something (e.g., reliable server push) that stateless StreamableHTTP does not support.

## Examples

### Weak input

"Make my GitHub MCP server use StreamableHTTP."

### Skill-guided interaction

Q: What is this server's connection target?
GUESS: Given "GitHub MCP server," the target is the GitHub API — confirm or correct.

Q: Python or TypeScript?

Q: Will this run as a single local/dev instance, or hosted behind a load balancer, across multiple replicas, or on serverless/edge infrastructure?

Q: Does any tool need to push updates back to the client mid-session — sampling, progress notifications, or resource-subscription updates?

### Final output shape

> Recommended: `stateless_http=True`, `json_response=True`. Reason: [tied to the user's deployment-shape and server-push answers].
>
> [One complete, runnable server file in the confirmed SDK/language.]

## Red Flags

- Setting both flags to a copied default (e.g., always `True/True`) without asking the four required questions.
- Proceeding with a stateless scaffold after the user answered "yes" to server-push requirements.
- Generating stdio code under this skill.
- Re-asking a question the user already answered earlier in the project.
- Writing test code, deployment config, or optimization suggestions as part of this skill's output.
- Using protocol jargon in the recommendation instead of plain language.

## Verification

Before treating the output as complete, confirm:

- [ ] All four required inputs were gathered or reused from existing context, not skipped.
- [ ] `stateless_http` and `json_response` each have a stated reason tied to a specific answer.
- [ ] If server-push was needed, the skill stopped and redirected instead of scaffolding stateless mode anyway.
- [ ] The user confirmed the recommendation before code was generated.
- [ ] The scaffold is complete and runnable in the stated SDK/language.
- [ ] No test, deployment, or optimization content was included.
- [ ] Output is exactly the recommendation plus the scaffold — nothing else.

See `streamable-http-reference.md` for the decision matrix, the disabled-capability list, and the Python/TypeScript code patterns.
