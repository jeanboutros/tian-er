---
name: context7-docs
description: "Fetch up-to-date, version-specific library and API documentation using Context7. Triggered when any agent needs library docs, API references, setup instructions, or code examples. Tries the CLI first, falls back to MCP tools, and finally to direct URL fetch. Prevents hallucinated APIs and outdated code based on stale training data."
---

# Context7 — Library Documentation Lookup

## Purpose

Fetch current, version-specific documentation for any library or API directly into the agent's context. This skill prevents:
- Hallucinated APIs that don't exist
- Outdated code examples from training data
- Generic answers for old package versions

## When to Trigger

- Agent needs to use a library API it hasn't verified against the source
- Agent is generating code that imports a third-party package
- Agent is writing setup, configuration, or installation steps
- Agent is answering a question about a library's behaviour or options
- User appends `use context7` or `use library /org/repo` to a prompt
- Any task where the Online Validation Rule in AGENTS.md applies to library docs

**Do NOT trigger** for project-internal code, first-party modules, or when the user has already pasted the relevant docs into the conversation.

---

## Lookup Protocol

Follow these steps in order. Move to the next step only if the current one fails.

### Step 1 — CLI (preferred)

Requires `ctx7` CLI installed (`npm install -g ctx7` or via `npx ctx7`).

**1a. Resolve the library ID:**

```bash
npx ctx7 library <library-name> "<query>"
```

Example:
```bash
npx ctx7 library next.js "app router middleware"
```

This returns a list of matching libraries with their Context7 IDs (e.g. `/vercel/next.js`).

**1b. Fetch documentation:**

```bash
npx ctx7 docs <libraryId> "<query>"
```

Example:
```bash
npx ctx7 docs /vercel/next.js "middleware authentication JWT"
```

This returns the relevant documentation sections, code examples, and API references.

**If the CLI is not installed or fails:** proceed to Step 2.

### Step 2 — MCP Tools (fallback)

If an MCP server for Context7 is configured (server URL: `https://mcp.context7.com/mcp`), use the MCP tools:

**2a. Resolve library ID:**

Call `resolve-library-id` with:
- `libraryName`: the library name (e.g. `"next.js"`)
- `query`: the user's question or task

**2b. Fetch documentation:**

Call `query-docs` with:
- `libraryId`: the resolved ID from step 2a (e.g. `/vercel/next.js`)
- `query`: the specific question to get relevant docs for

**If MCP tools are not available:** proceed to Step 3.

### Step 3 — Direct URL Fetch (last resort)

Fetch the library's official documentation URL directly using the agent's web fetch capability:
1. Identify the library's canonical docs URL (e.g. `https://nextjs.org/docs`)
2. Fetch the specific page relevant to the query
3. Extract the relevant API signatures, code examples, and configuration

**If the URL is unreachable:** state explicitly that the documentation could not be verified and ask the user for guidance. Do NOT guess from training data.

---

## Setup

### CLI Setup (recommended)

Requires Node.js 18+:

```bash
npx ctx7 setup
```

This authenticates via OAuth, generates an API key, and installs the skill or MCP server for the active agent. Use `--cursor`, `--claude`, or `--opencode` to target a specific agent.

To remove: `npx ctx7 remove`

### MCP Setup (alternative)

If the CLI setup fails or is not suitable, configure the MCP server manually:

1. Get a free API key from [context7.com/dashboard](https://context7.com/dashboard)
2. Add to your MCP client configuration:

```json
{
  "mcpServers": {
    "context7": {
      "url": "https://mcp.context7.com/mcp",
      "headers": {
        "CONTEXT7_API_KEY": "<your-api-key>"
      }
    }
  }
}
```

For OpenCode, add to `~/.config/opencode/config.json`. For Cursor, add to `.cursor/mcp.json`. For Claude Code, use `/mcp add`.

---

## Available Tools Reference

| Mode | Command / Tool | Input | Output |
|------|---------------|-------|--------|
| CLI | `ctx7 library <name> <query>` | Library name + search query | List of matching libraries with Context7 IDs |
| CLI | `ctx7 docs <libraryId> <query>` | Context7 library ID + query | Documentation sections, code examples, API refs |
| MCP | `resolve-library-id` | `libraryName` + `query` | Context7-compatible library ID |
| MCP | `query-docs` | `libraryId` + `query` | Documentation for the specified library |

### Library ID Format

Context7 IDs use the `/org/repo` format:
- `/vercel/next.js`
- `/supabase/supabase`
- `/mongodb/docs`
- `/expressjs/express`

If you already know the library ID, skip the resolve step and go directly to `ctx7 docs` or `query-docs`.

---

## Usage Patterns

### In prompts
- `use context7` — agent resolves the library from context and fetches docs
- `use library /supabase/supabase` — agent fetches docs for a specific library directly

### Version-specific queries
Mention the version in the query and Context7 will match it:
```
How do I set up Next.js 14 middleware? use context7
```

---

## Rules

1. **Always verify before generating.** If the task involves a third-party library, fetch its docs before writing code — don't rely on training data.
2. **Follow the fallback chain.** CLI → MCP → Direct URL. Never skip to guessing.
3. **One library at a time.** Resolve and fetch docs for each library separately.
4. **State failures explicitly.** If all three methods fail, tell the user: "Could not verify [library] documentation. Please provide the relevant docs or confirm the API."
5. **Don't block on setup.** If Context7 is not installed, use Step 3 (direct fetch) and proceed. Suggest setup to the user for future sessions.

---

## Source

- Repository: [github.com/upstash/context7](https://github.com/upstash/context7)
- Website: [context7.com](https://context7.com/)
- CLI package: [ctx7 on npm](https://www.npmjs.com/package/ctx7)
- MCP package: [@upstash/context7-mcp on npm](https://www.npmjs.com/package/@upstash/context7-mcp)

## Self-Reflection Clause

After any issue caused by outdated or incorrect library usage:
1. **Was the library doc fetched before generating code?** — If not, this skill was not triggered when it should have been.
2. **Which fallback step was used?** — If Step 3 or guessing, suggest Context7 setup to the user.
3. **Update the knowledge base** — Note the library ID for future lookups to skip the resolve step.
