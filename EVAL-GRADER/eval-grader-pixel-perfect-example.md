<example>

<input>

  <prompt>
Summarize the support ticket below into JSON with fields: customer_name,
issue_summary (<=20 words), priority (low/medium/high), and
suggested_next_step. Return only valid JSON, no other text.

Ticket: "Hi, this is J. Alvarez. Every time I try to upload a profile photo
larger than 5MB on the mobile app, it crashes immediately. This has happened
three times today and I can't update my account. Please help soon."
  </prompt>

  <output>
{"customer_name": "J. Alvarez", "issue_summary": "Mobile app crashes when uploading profile photos larger than 5MB", "priority": "medium", "suggested_next_step": "Escalate to mobile engineering to investigate upload size handling and provide a temporary workaround to the customer"}
  </output>

</input>

<grading_output>
Score: 15/20

The JSON is well-formed, contains only the object as instructed, and all four
required fields are present with no invented details; customer_name and
priority are both reasonable reads of the ticket. Two things hold this back
from a higher score: issue_summary runs to 11 words so it's within the
20-word cap, but suggested_next_step is written as a 17-word sentence when
the rest of the schema implies short, structured values, making it visually
inconsistent with the other fields, and "medium" priority is a defensible but
debatable read of a customer blocked from completing account setup, which a
tighter prompt would resolve. The highest-leverage fix is to add explicit
length or format guidance for suggested_next_step (e.g. "5-8 words, action
plus owner") and a one-line priority rubric (e.g. "high = blocks a core
workflow") so the model isn't inferring severity unassisted.
</grading_output>

<why_this_is_pixel_perfect>
- Output format matches the skill's <output_format> template exactly: a
  "Score: X/20" line, one blank line, then exactly one paragraph.
- The paragraph names concrete strengths (schema compliance, no invented
  fields), concrete weaknesses (inconsistent field style, debatable priority
  call), and ends with one specific, actionable prompt fix, not a vague
  "improve clarity" note.
- No headers, bullets, multi-paragraph breakdown, or hedging language appear,
  per the skill's <constraints> block.
- The score sits inside the correct band (13-16, "solid") and is justified
  against the nearest boundary rather than defaulting to the midpoint.
- No hard-failure conditions apply, so no cap was triggered, and the response
  says nothing about rewriting the prompt since that wasn't requested.
</why_this_is_pixel_perfect>

</example>
