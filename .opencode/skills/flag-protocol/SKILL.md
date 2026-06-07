---
name: flag-protocol
description: "Structured flag protocol for non-PM agents to raise decisions, tickets, clarifications, and advisories. All agents must use this format when they identify work that requires PM attention."
---

# Flag Protocol

## Purpose

When any agent identifies work that needs a task, clarification, decision, or advisory but doesn't have the authority to create it, they produce a **flag** rather than creating the artifact themselves. Only `@pm` processes flags into actual tasks.

## When to Use

- An agent discovers a bug or technical debt that needs tracking
- An agent encounters ambiguity that needs user input (clarification)
- An agent identifies a design choice with risk (decision needed)
- An agent finds a non-blocking issue worth tracking (advisory)

## Format

Every flag MUST use this exact format:

```markdown
## Flag: [type] — [short title]

| Field | Value |
|-------|-------|
| Type | `task` / `clarification` / `decision` / `advisory` |
| Priority | `critical` / `high` / `medium` / `low` |
| Raised by | Agent role (e.g. "Code Architect", "Hardware Engineer") |
| Blocking | `yes` / `no` — does this block the current pipeline step? |
| Reference | Current task or phase |

## Description
What was found and why it needs attention.

## Evidence
Code snippets, datasheet references, or PoC.

## Suggested action
What the flagging agent recommends.
```

## Routing Rules

| Blocking? | Route | Effect |
|-----------|-------|--------|
| Yes | Supreme Leader pauses pipeline | Presented to user immediately |
| No | Queued for PM | PM creates task in next planning cycle |
| Advisory | Logged | Persisted for future reference |

## Rules

1. **Only PM creates tasks** — all other agents raise flags
2. **One flag per issue** — don't bundle unrelated findings
3. **Be specific** — "buffer overflow in read_payload line 42" not "potential safety issue"
4. **Include evidence** — code, datasheet page, or build output
5. **Blocking flags halt the pipeline** — use sparingly, only for true blockers

## Self-Reflection Clause

After fixing any bug or resolving any issue that required debugging, you MUST ask:
1. **Why was this bug missed?** — What review, test, or protocol gap allowed it through?
2. **What procedural safeguard would have caught it?** — What specific check, test, or verification step would have prevented it?
3. **Update the knowledge base** — Add the lesson to the relevant domain skill (e.g., nrf24l01plus SKILL.md for nRF24 hardware bugs, or the appropriate learning doc in `docs/learning/`) so the same class of bug is caught earlier next time.
