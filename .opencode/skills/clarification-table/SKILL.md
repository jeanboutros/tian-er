---
name: clarification-table
description: "Generate an interactive HTML table from blocked agent clarification questions. Use this skill when an agent has accumulated multiple STATUS: BLOCKED questions from the assumption-trap protocol and the user wants to batch-answer them in a web browser rather than one at a time."
---

# Clarification Table

## Purpose

When an agent hits multiple blocking questions during the assumption-trap protocol, answering them one at a time in the command line becomes tedious. The Clarification Table batches all pending questions into a single self-contained HTML file that the user opens in a browser, answers quickly with radio buttons and custom text inputs, and saves as JSON. The agent then reads the saved JSON and resumes the pipeline with all answers resolved.

This skill does NOT replace the assumption-trap protocol — it supplements it. Agents still follow the assumption-trap format (`STATUS: BLOCKED`, `QUESTION`, `OPTIONS`, `IMPACT`). The Clarification Table is the batching mechanism that makes it fast for the user.

## When to Trigger

- **Multiple blocked questions exist** — 3 or more `STATUS: BLOCKED` halts from any agent
- **User requests batching** — "just give me all the questions at once" or similar
- **Pipeline is stalled** — the dispatcher detects multiple unresolved clarification points and needs to unblock progress
- **Cross-agent coordination** — questions from multiple agents (SW Engineer, HW Engineer, Wireless Expert) need answers before any can proceed

## How to Use

### Step 1: Collect Pending Questions

Gather all `STATUS: BLOCKED` outputs from the current session. Each question must have:

| Field | Description |
|-------|-------------|
| `id` | Unique identifier (`q1`, `q2`, `q3`, ...) |
| `question` | The precise question from the `QUESTION:` field of the assumption-trap output |
| `context` | The analysis context from the `CONTEXT:` field |
| `recommended` | The agent's recommended answer (based on best judgment, datasheet, or spec) |
| `options` | Concrete choices from the `OPTIONS:` field (may include "Option A", "Option B", ...) |
| `impact` | `HIGH`, `MEDIUM`, or `LOW` — from the `IMPACT:` field |
| `source` | Datasheet page, spec section, or authoritative URL |

### Step 2: Generate the HTML File

1. Read `template.html` from this skill directory
2. Replace the placeholder `___QUESTIONS_JSON___` with the actual JSON array of question objects
3. Set the `TICKET_ID` and `SOURCE_AGENT` variables at the top of the script block
4. Write the modified HTML to a file (e.g., `/tmp/clarification-table.html` or a location the user specifies)
5. **Do not modify any other part of the template** — the template is the source of truth for behavior and styling

### Step 3: Instruct the User

Tell the user:

```
Open file:///tmp/clarification-table.html in your browser.
Answer each question, then click "Save Answers".
This will download `clarification-answers.json`.
Return the file path or paste the JSON contents back to me.
```

### Step 4: Process Answers

When the user returns the JSON:

1. Parse the answers array
2. For each entry, look up `id` → `choice` / `custom`
3. Feed the resolved answers back into the pipeline
4. Unblock the stalled agents with the resolved values
5. Continue the pipeline with Phase A design or Phase B build

### Step 5: Clean Up (Optional)

Remove the generated HTML file after the session if the user does not need it preserved.

## Data Injection Convention

The template uses a single placeholder for data injection:

```
const QUESTION_DATA = ___QUESTIONS_JSON___;
```

**Before injection:**
```javascript
const QUESTION_DATA = ___QUESTIONS_JSON___;
```

**After injection:**
```javascript
const QUESTION_DATA = [
  {
    "id": "q1",
    "question": "What is the reset value of RF_CH bit 7?",
    "context": "Analyzing nRF24L01+ register map, RF_CH register field at bit 7",
    "recommended": "Write 0 — reserved bit, datasheet says must be 0",
    "options": [
      "Option A: Write 0 (as per datasheet §6.4)",
      "Option B: Write 1 (some errata mention alternate behavior)"
    ],
    "impact": "HIGH",
    "source": "nRF24L01+ datasheet page 34, §6.4"
  }
];
```

The agent performing the injection MUST use valid JSON. The placeholder is literal text `___QUESTIONS_JSON___` — not a delimited string, not a quoted string, not inside backticks. Just the token itself.

## Question JSON Format

Each question object in the array:

```json
{
  "id": "string (unique identifier)",
  "question": "string (the precise question)",
  "context": "string (what you were analyzing)",
  "recommended": "string (agent's recommended answer with reasoning)",
  "options": ["string", "string", ...],
  "impact": "HIGH | MEDIUM | LOW",
  "source": "string (datasheet page, spec section, URL)"
}
```

## Answer JSON Format (Output)

The user's browser download produces:

```json
{
  "ticket": "string (TASK-ID)",
  "generated_at": "string (ISO 8601 timestamp)",
  "source_agent": "string (agent name)",
  "answers": [
    {
      "id": "string",
      "question": "string",
      "choice": "recommended | option_0 | option_1 | option_2 | option_3 | option_4 | custom",
      "custom": "string | null"
    }
  ]
}
```

## Template

The HTML template lives at `template.html` in this skill directory. It is a zero-dependency, self-contained file that works offline when opened directly from the filesystem.

## Constraints

- **Zero dependencies** — no CDN, no npm, no fonts from Google. System font stack only.
- **Offline-first** — works from `file://` protocol, no server required
- **Single file** — all CSS inline in `<style>`, all JS inline in `<script>`
- **No build step** — just open `template.html` in any modern browser to see sample data
- **No data persistence** — answers are only saved when the user clicks "Save Answers". No localStorage, no server POST. Explicit download only.

## Edge Cases

- **Questions with 0 options** — show only "Recommended" and "Custom" radio buttons
- **Questions with 1 option** — show "Recommended", "Option A", and "Custom"
- **Questions with 5+ options** — all options rendered dynamically with option_0 through option_N keys
- **Empty custom text** — if user selects "Custom" but leaves text empty, Save button shows validation error
- **No questions** — if `QUESTION_DATA` is an empty array, show a "No pending questions" message
- **Malformed JSON** — the template file itself (with sample data) should always render. Only injected data needs to be valid.

## Security

- The HTML file is generated by a trusted agent and opened locally by the user
- No data leaves the user's machine — the Blob download is local only
- No XSS vector: question data is injected as a JSON literal, not through `innerHTML`
- The template uses `textContent` for rendering, not `innerHTML`, except for the generated download Blob

## Related Skills

- **assumption-trap** — the protocol that generates the `STATUS: BLOCKED` questions this skill batches
- **pipeline** — the pipeline state machine that stalls when questions are unresolved
- **pipeline-passport** — tracks which questions have been resolved
