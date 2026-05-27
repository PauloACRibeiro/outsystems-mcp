---
description: Send feedback about your OutSystems agent experience to the AI Platform team
argument-hint: <your feedback>
---

The user typed `/feedback $ARGUMENTS` — they want to report something about the OutSystems agent experience (a bug, a thumbs-up/down, a comment about a tool that misbehaved, etc.).

Call the OutSystems `submit_feedback` MCP tool with:
- `name`: `"user_feedback"`
- `value`: `"$ARGUMENTS"` (the user's message verbatim — keep it short; this is the categorical/summary)
- `rationale`: `"$ARGUMENTS"` (same content; the rationale field is the longer free-text body)
- `mentor_session_id`: if there's a `mentor_session_id` you've been working with in this conversation, include it so the AI Platform team can co-locate the feedback with the relevant mentor trace. Otherwise omit it — the server falls back to a standalone feedback record.

If the user said something very long (multi-paragraph), put a one-line summary in `value` (e.g., `"bug-report"`, `"feature-request"`, `"thumbs-down"`) and the full text in `rationale`. Strip any code snippets, OML, full transcripts, or secrets from `rationale` — replace with `[redacted]` and tell the user you did so.

After the call returns `status: "accepted"`, tell the user something like "Thanks — feedback sent to the AI Platform team." If it returns `status: "not_configured"`, the feedback infrastructure isn't enabled on this stamp; tell the user "Feedback isn't configured on this OutSystems environment yet; I've noted what you said but it didn't reach the team."

Don't volunteer this command. Don't proactively ask "would you like to submit feedback?" — only run it when the user explicitly invokes it OR uses the `submit_feedback` MCP tool yourself for `agent_observation` per the SKILL.md guidance.
