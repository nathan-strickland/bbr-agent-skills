# Conversational Gate Protocol

---
name: conversational-gate-protocol
description: >
  Response contract for specialist agents that run inside SuperFlows and
  converse with users through HITL gates. Enforces the result-or-needs_input
  JSON protocol and explicit state carrying across stateless invocations.
version: 0.1.0
parameters: none
---

## Purpose

Agents inside a SuperFlow cannot chat. Every human interaction happens by
pausing the flow at a HITL gate that carries a question out, and resuming with
a structured answer. This skill defines the response contract that makes that
work. Attach it to any agent that needs to ask users questions from inside a
flow. The agent's own instructions define its domain; this skill defines only
the protocol.

## The contract

Every invocation of you is FRESH: you have no memory of previous passes. All
continuity comes from the state object described below.

Inputs you may receive:
- First pass: the flow's trigger fields (e.g. `request_text`).
- Loop passes: `{state, user_answer, answered_gate_id}`.

You return ONLY JSON, exactly one of the two shapes below. Never respond in
prose. Clarifying questions are NOT an exception: a question is a
`needs_input` response, never free text.

### Shape 1: result (your task is complete)

```json
{"status": "result", ...domain fields defined by your own instructions...}
```

### Shape 2: needs_input (you require something from the user)

```json
{"status": "needs_input",
 "state": {
   "request_text": "<original request>",
   ...every field you have established so far...,
   "candidates": [{"id": "...", "label": "..."}]
 },
 "gate": {
   "gate_id": "<short stable id for this question>",
   "question": "<one clear question>",
   "context": "<what you understood so far, one line>",
   "answer_schema": {
     "type": "choice" | "value",
     "options": [{"id": "...", "label": "..."}],
     "expects": "<format hint, for value type>",
     "allow_free_text": true
   }
 }}
```

`options` is required for `choice`; `expects` is required for `value`.
Include a candidates list in `state` only if you have already searched.

### Shape discipline (exactly one shape per turn)

- When `status` = `result`: `gate` MUST be empty (`gate_id` "", `question` "",
  `options` []). Populated domain fields, empty gate.
- When `status` = `needs_input`: `gate` MUST be fully populated. Never both.

### Identifiers require the user

`status` = `result` is permitted ONLY when the identifying answer came from
the user: they confirmed a candidate in a previous turn (your input carries
`answered_gate_id` and their `user_answer`), or they chose the "new" option
and you minted via your tool. A strong or even exact search match is NEVER
sufficient on its own: return `needs_input` with the confirmation gate.

## State rules

1. `state` must contain EVERYTHING needed to continue without re-doing work:
   the fields established so far, and any candidate list (ids and labels) from
   searches already run. It is a snapshot, not a transcript: no tool dumps, no
   prose history.
2. If `state` is present in your input: do NOT re-interpret the original
   request and do NOT re-run searches you already ran. Continue from `state`,
   applying `user_answer` as the reply to `answered_gate_id` (for example, map
   a chosen candidate id to its identifier; merge a supplied value into the
   fields).
3. Only you read or write `state`. The flow, the gates and the client carry it
   through untouched.

## Question discipline

- Ask ONE question per gate. If several fields are missing, ask for them in a
  single `value` gate ("Please give the vintage and bottle size") rather than a
  gate per field.
- Prefer `choice` gates with explicit options wherever the answers are
  enumerable; reserve `value` gates for genuinely open inputs.
- Never choose on the user's behalf. A weak or empty search result is a reason
  to ASK (or to offer a "none of these / new" option), not to guess.
- Never invent identifiers, codes or values; they come only from your tools or
  from the user's answers.
