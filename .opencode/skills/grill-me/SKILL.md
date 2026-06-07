---
name: grill-me
description: "Adversarial design review through relentless questioning. Used for the Dual-Model Challenge — stress-tests plans and designs by walking down every branch of the decision tree until shared understanding is reached."
---

# Grill Me — Adversarial Design Review

## Purpose

Interview the design/implementation relentlessly about every aspect until all assumptions are exposed. Walk down each branch of the decision tree, resolving dependencies between decisions one-by-one.

## When to Use

- **Dual-Model Challenge** in Phase A (architecture review)
- **Dual-Model Challenge** in Phase C (verification)
- **Pre-implementation review** of complex features
- **Architecture validation** before committing to a design

## The Process

1. **Read the design/code thoroughly**
2. **For each decision**, ask:
   - Why this approach over alternatives?
   - What happens at the boundary/edge cases?
   - What does the datasheet say about this specific case?
   - What happens if the hardware doesn't behave as expected?
3. **Walk decision trees branch-by-branch:**
   - If this field is 0xFF, what happens in `from_byte()`?
   - If SPI returns all zeros, how does the driver detect failure?
   - If CE is never asserted, what state is the radio in?
4. **Resolve dependencies:**
   - Does this design depend on assumptions about another component?
   - What if that component changes its interface?

## Question Categories for Embedded

### Register Design
- What happens with reserved bits? (Read as 0? Write as 0? Write as 1?)
- What if from_byte gets an illegal encoding for an enum field?
- Are the bit positions verified against the datasheet? Which page?

### Protocol Correctness
- Is the byte order correct? (MSB-first vs LSB-first)
- Is the bit order correct? (BLE is LSbit-first, nRF24 is MSbit-first)
- What whitening seed is used? Is it channel-dependent?

### Safety
- What's the maximum buffer size? Can it overflow?
- What's the stack depth of this task? Measured or estimated?
- What happens if SPI fails mid-transfer?

### Platform Independence
- Would this compile without ESP-IDF installed?
- Does any library header pull in platform-specific types?
- Is the HAL interface sufficient for a different platform?

## Rules

- Ask questions ONE AT A TIME
- If a question can be answered by reading the code/datasheet, do that first
- Provide your recommended answer with each question
- Don't stop until every branch of the decision tree is resolved
- Be thorough but fair — critique real issues, not style preferences

## Self-Reflection Clause

After fixing any bug or resolving any issue that required debugging, you MUST ask:
1. **Why was this bug missed?** — What review, test, or protocol gap allowed it through?
2. **What procedural safeguard would have caught it?** — What specific check, test, or verification step would have prevented it?
3. **Update the knowledge base** — Add the lesson to the relevant skill or learning doc in `docs/learning/` so the same class of bug is caught earlier next time.
