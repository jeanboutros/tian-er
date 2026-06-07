---
name: brainstorming
description: "Structured design dialogue that validates ideas before implementation begins. Enforces a hard gate: no code until design is presented and approved by all specialists."
---

# Brainstorming Ideas Into Designs

## Overview

Help turn ideas into fully formed designs through structured collaborative dialogue. Start by understanding the context, then ask questions to refine, present the design, and get approval.

## The Hard Gate

```
NO CODE, SCAFFOLDING, OR IMPLEMENTATION UNTIL DESIGN IS APPROVED
```

Even "simple" tasks go through this process. Simple tasks are where unexamined assumptions cause the most wasted work.

## The Process

### Step 1: Context Exploration
- What existing code is affected?
- What hardware capabilities/constraints apply?
- What does the datasheet say about this?

### Step 2: Clarifying Questions (One at a Time)
- Ask the FIRST question that would change the design
- Wait for answer
- Ask the next question
- Continue until design is unambiguous

### Step 3: Design Proposal

```markdown
## Design: [Feature Name]

### Goal
[One sentence — what this achieves]

### Approach
[How it will work — component boundaries, data flow, register usage]

### Files to Create/Modify
| File | Action | Purpose |
|------|--------|---------|
| [path] | create/modify | [what and why] |

### API Surface
[New public types, functions, constants]

### Hardware Considerations
[Registers involved, timing constraints, pin usage]

### Acceptance Criteria
1. [Binary pass/fail]
2. [Binary pass/fail]

### Risks
[What could go wrong — or "None identified"]
```

### Step 4: Specialist Review
Present design to ALL specialists for input:
- SW Engineer: architecture and API
- HW Engineer: register and timing correctness
- Wireless Expert: protocol compliance
- Security: safety and bounds checking
- Test Engineer: testability
- Docs Writer: documentation needs

### Step 5: Approval Gate
ALL specialists must approve before implementation begins.

## Anti-Pattern: "This Is Too Simple To Need A Design"

Every change goes through this process. A register header, a utility function, a config change — all of them. The design can be SHORT (a few sentences for truly simple things), but it MUST be presented and approved.
