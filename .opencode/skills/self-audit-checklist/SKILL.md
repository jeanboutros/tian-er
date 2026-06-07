---
name: self-audit-checklist
description: "Mandatory self-audit protocol for all reviewing agents. Before issuing any verdict in Phase C, agents MUST complete this checklist explicitly. If any row is missing, the review is INVALID."
---

# Self-Audit Checklist

## Purpose

Prevent reviewers from focusing on one dimension and missing others. Every Phase C review MUST cover ALL dimensions explicitly.

## When to Use

- **All specialist agents** — Before issuing APPROVED/REJECTED in Phase C
- **Code Architect** — Before declaring a task complete

## Mandatory Checklist

Before writing your verdict, complete this checklist **explicitly in your output**. Every row must show a concrete finding or "PASS — checked [evidence]".

```markdown
### Self-Audit Checklist

| Category | Checked? | Finding or PASS |
|----------|----------|-----------------|
| Build passes (exit 0, no warnings) | yes/no | [output evidence] |
| Typed enums / vocabulary types (no raw integers in API) | yes/no | [files checked] |
| Documentation on new public symbols | yes/no | [symbols checked] |
| Spec/datasheet fidelity (fields match spec) | yes/no | [reference cited] |
| Module boundary (no platform headers in shared modules) | yes/no | [includes checked] |
| Reserved/padding fields handled | yes/no | [serialisation checked] |
| No magic numbers in doc examples | yes/no | [examples checked] |
| Buffer safety (bounded copies) | yes/no | [buffers checked] |
| AGENTS.md compliance | yes/no | [rules verified] |
| Conventional commit ready | yes/no | [message format] |
```

## How to Complete Each Row

### Build passes
- Evidence: actual build output showing success with exit code 0
- If not run in this session, state "NOT VERIFIED — requires build"

### Typed vocabulary
- For every public method: if a parameter has a finite set of legal values, verify the parameter TYPE enforces this at compile time, not just through naming conventions
- Named constants used as raw-typed parameters are documentation aids, NOT type safety — a raw-typed parameter accepts any value
- Check that raw overloads are `private` or `protected` where typed alternatives exist

### Documentation on public symbols
- List every new public symbol (function, struct, class, enum)
- Verify each has appropriate documentation (`@brief`, `@param`, `@return` or equivalent for your doc tool)

### Spec/datasheet fidelity
- For each protocol field or register struct: cite the spec/datasheet page where the layout is defined

### Module boundary
- Check `#include` or `import` lines in all shared/library public headers
- No platform-specific headers should leak into portable/shared modules

### Reserved/padding fields
- Check serialisation: reserved fields are masked/zeroed on write and ignored on read

### No magic numbers in doc examples
- Scan code examples in documentation for unexplained raw literals
- All values in examples should use named constants or typed vocabulary

### Buffer safety
- All `memcpy`, array access, and I/O transfers have bounded size
- No unbounded reads from external data

### AGENTS.md compliance
- Cross-check against the rules in AGENTS.md
- Verify commit message format, learning docs policy, etc.

## Self-Reflection Clause

After fixing any bug or resolving any issue that required debugging, you MUST ask:
1. **Why was this bug missed?** — What review, test, or protocol gap allowed it through?
2. **What procedural safeguard would have caught it?** — What specific check, test, or verification step would have prevented it?
3. **Update the knowledge base** — Add the lesson to the relevant skill or learning doc in `docs/learning/` so the same class of bug is caught earlier next time.
