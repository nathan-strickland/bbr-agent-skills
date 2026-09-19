# Conversational Gate Protocol

---
name: conversational-gate-protocol
description: >
  Response contract for specialist agents that run inside SuperFlows and
  converse with users through HITL gates. Enforces the result-or-needs_input
  JSON protocol and explicit state carrying across stateless invocations.
version: 0.2.0
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
- First pass: the flow's trigger fields.
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
   "...every field you have established so far...": "...",
   "candidates": [{"value": "...", "identifier": "...", "description": "..."}]
 },
 "gate": {
   "gate_id": "<short stable id for this question>",
   "question": "<one clear question>",
   "summary": "<what you understood so far, one line>",
   "subject": {"identifier": "<the thing being decided about>", "description": "<what kind of thing>"},
   "pre_filled": "<the option value or value you propose, or null>",
   "answer_schema": {
     "type": "choice | value",
     "options": [{"value": "...", "identifier": "...", "description": "..."}],
     "expects": "<format hint for value type, else null>"
   }
 }}
```

`options` is required for `choice` and must be `[]` for `value`. `expects` is
required for `value` and must be `null` for `choice`. Include a candidates
list in `state` only if you have already searched.

`subject` when one identifier is the thing being decided about; option identifiers when each choice names a different one. Null when neither applies. The collision gate's subject is null, which is
  correct - its identifiers are per-option.

### Shape discipline (exactly one shape per turn)

- When `status` = `result`: `gate` MUST be empty - `gate_id` "", `question` "",
  `summary` "", `pre_filled` null, `options` []. Populated domain fields,
  empty gate.
- When `status` = `needs_input`: `gate` MUST be fully populated. Never both.

### Identifiers require the user

`status` = `result` is permitted ONLY when the identifying answer came from
the user: they confirmed a candidate in a previous turn (your input carries
`answered_gate_id` and their `user_answer`), or they chose a "none of these"
option and you minted via your tool. A strong or even exact search match is
NEVER sufficient on its own: return `needs_input` with the confirmation gate.

### Proposing is not choosing

`pre_filled` carries your best proposal so the person reviews rather than
starts from nothing, and so the difference between what you proposed and what
they gave is recorded as a correction.

- It NEVER permits `status` = `result`. The rule above is unchanged: a
  proposal is not a confirmation, however confident you are.
- Set it to `null` when you genuinely have no proposal. Never guess in order
  to populate it - an honest absence is worth more than a fabricated default.
- For `choice`, it is one of the `value`s in `options`. For `value`, it is a
  value conforming to `expects`.

### Options carry their parts, never a composed label

Each option has three fields and you populate them separately:

- `value` - what is returned when the person picks it. Short, stable, machine
  readable.
- `identifier` - the business code or reference this option names, or `null`
  where the option names none.
- `description` - human text. It MUST NOT repeat the identifier.

Do not build a display string. The client composes the label from the parts
and guarantees the identifier appears exactly once, which it cannot do if you
have already embedded it. If an identifier belongs to an option, it goes in
`identifier`, not in `question`, `summary` or `description`.

## State rules

1. `state` must contain EVERYTHING needed to continue without re-doing work:
   the fields established so far, and any candidate list from searches already
   run. It is a snapshot, not a transcript: no tool dumps, no prose history.
2. If `state` is present in your input: do NOT re-interpret the original
   request and do NOT re-run searches you already ran. Continue from `state`,
   applying `user_answer` as the reply to `answered_gate_id`.
3. Only you read or write `state`. The flow, the gates and the client carry it
   through untouched.

## Question discipline

- Ask ONE question per gate. If several fields are missing, ask for them in a
  single `value` gate rather than a gate per field.
- Prefer `choice` gates with explicit options wherever the answers are
  enumerable; reserve `value` gates for genuinely open inputs.
- Never choose on the user's behalf. A weak or empty search result is a reason
  to ASK, or to offer a "none of these" option, not to guess.
- Never invent identifiers, codes or values; they come only from your tools or
  from the user's answers.

## What you do not set

These are decided outside you and you must not emit them:

- whether free text is accepted for a gate
- which role may answer a gate, and whether the answerer must differ from the
  initiator
- who is notified
- your own identity

They are gate policy and flow configuration. Emitting them would let wording
vary a control, which is what this contract exists to prevent.
