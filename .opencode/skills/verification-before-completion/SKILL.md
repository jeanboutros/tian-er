---
name: verification-before-completion
description: "Enforce running the actual verification command before claiming work is complete. No success assertions without fresh evidence from the build or test run."
---

# Verification Before Completion

## Overview

Claiming work is complete without verification is dishonesty, not efficiency.

Core principle: **Evidence before claims, always.**

## The Iron Law

```
NO COMPLETION CLAIMS WITHOUT FRESH VERIFICATION EVIDENCE
```

If you haven't run the verification command in this message, you cannot claim it passes.

## The Gate Function

Before ANY of these statements, you MUST have fresh evidence:

- "Build passes" → requires build tool output showing successful compilation
- "Tests pass" → requires test runner output showing all tests pass
- "Bug is fixed" → requires build + test/runtime verification showing correct behaviour
- "Task is complete" → requires ALL acceptance criteria evidenced
- "No warnings" → requires build output with no warning lines

## The Five Steps

1. **Identify** the proof command: your project's build/test command (e.g. `make`, `cmake --build`, `npm test`, `cargo test`)
2. **Run it fresh** — not from a previous session, not from memory
3. **Read full output** — check exit code AND scan for warnings/errors
4. **Verify the claim matches** — build success output for your toolchain
5. **State the result with evidence** — quote the relevant output line

## What Counts as Evidence

| Claim | Required Evidence |
|-------|-------------------|
| "Build passes" | Terminal output showing build succeeded with exit code 0 |
| "No warnings" | grep output showing zero warning lines |
| "Tests pass" | Test runner output showing all tests pass |
| "Feature works" | Runtime or integration test output confirming behaviour |

## What Does NOT Count

| Not Evidence | Why |
|-------------|-----|
| "I believe it will build" | Not verified |
| "Based on the code, it should work" | Not verified |
| "It compiled last time" | Stale — changes since then |
| "The logic is correct" | Correctness ≠ compilation |
| Agent confidence | Irrelevant without proof |

## Violations

If you catch yourself about to claim success without evidence:
1. STOP
2. Run the verification command
3. THEN make the claim with evidence attached

## Self-Reflection Clause

After fixing any bug or resolving any issue that required debugging, you MUST ask:
1. **Why was this bug missed?** — What review, test, or protocol gap allowed it through?
2. **What procedural safeguard would have caught it?** — What specific check, test, or verification step would have prevented it?
3. **Update the knowledge base** — Add the lesson to the relevant skill or learning doc in `docs/learning/` so the same class of bug is caught earlier next time.
