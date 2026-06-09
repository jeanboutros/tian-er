---
name: github
description: "GitHub operations for Tian'er: conventional commits, SSH authentication with agent socket, safe push, commit grouping rules. Loaded by Supreme Leader before any git commit or push dispatch."
---

# GitHub Operations

## Purpose

This skill defines the project's GitHub workflow. Every agent that performs git operations MUST load this skill first. It operationalizes the Commit Rules from `AGENTS.md` and adds Tian'er-specific SSH authentication and push safety rules learned from real failures in this project.

## When to Trigger

- **Loaded by Supreme Leader** before dispatching any commit or push task
- **Auto-triggered** when the user says "commit", "push", or any git operation
- Loaded alongside `verification-before-completion` — no push claim without fresh `git push` output

---

## Conventional Commits v1.0.0

This project follows the [Conventional Commits v1.0.0](https://www.conventionalcommits.org/en/v1.0.0/) specification. The spec was verified against the canonical source on 2026-06-09.

### Format

```
<type>[optional scope][!]: <description>

[mandatory body — what changed and why]

[optional footer(s)]
```

### Specification Rules (from the canonical spec)

1. Commits MUST be prefixed with a type (`feat`, `fix`, etc.), followed by OPTIONAL scope, OPTIONAL `!` for breaking changes, and REQUIRED colon and space.
2. `feat` — adds a new feature (correlates with MINOR in SemVer)
3. `fix` — patches a bug (correlates with PATCH in SemVer)
4. A scope MAY be provided in parentheses after the type: `feat(parser): ...`
5. A description MUST immediately follow the colon and space — short summary of changes
6. A body MAY be provided one blank line after the description, for additional context
7. The body is free-form, any number of paragraphs
8. Footers MAY be provided one blank line after the body, using `token: value` or `token #value` format (git trailer convention)
9. Footer tokens use `-` for whitespace (`Acked-by`, `Reviewed-by`). Exception: `BREAKING CHANGE` may also be used as a token
10. Footer values may contain spaces and newlines
11. Breaking changes MUST be indicated by `!` before the colon, OR by `BREAKING CHANGE:` footer
12. If footer: `BREAKING CHANGE: <description>` (uppercase, colon, space)
13. If `!` prefix: the description SHALL describe the breaking change. Footer is OPTIONAL
14. Types other than `feat` and `fix` MAY be used
15. Types are NOT case-sensitive (except `BREAKING CHANGE` which MUST be uppercase)
16. `BREAKING-CHANGE` is synonymous with `BREAKING CHANGE` as a footer token

### Tian'er Type Table

| Type | When to Use | SemVer |
|------|-------------|--------|
| `feat` | New feature — component, service, script, or capability | MINOR |
| `fix` | Bug fix in an existing file | PATCH |
| `docs` | Documentation-only changes (README, design docs, AGENTS.md) | — |
| `refactor` | Code restructuring with no behaviour change | — |
| `chore` | Maintenance (dependency updates, file moves, counter resets, skill updates) | — |
| `style` | Formatting, whitespace — no logic change | — |
| `test` | Adding or updating tests | — |
| `build` | Build system or external dependency changes | — |
| `ci` | CI configuration changes | — |
| `perf` | Performance improvement with no functional change | — |
| `revert` | Reverts a previous commit (body MUST reference the reverted SHA) | — |

### Real Examples (from the spec)

```text
# fix with no body
fix: prevent racing of requests

# Feature with scope
feat(lang): add Polish language

# Breaking change with ! prefix
feat!: drop support for Node 6

# Breaking change with footer
feat: allow provided config object to extend other configs

BREAKING CHANGE: `extends` key in config file is now used for extending other config files

# Multi-paragraph body with multiple footers
fix: prevent racing of requests

Introduce a request id and a reference to latest request. Dismiss
incoming responses other than from latest request.

Remove timeouts which were used to mitigate the racing issue but are
obsolete now.

Reviewed-by: Z
Refs: #123

# Revert
revert: let us never again speak of the noodle incident

Refs: 676104e, a215868
```

### Tian'er Scope Convention

Use the component or directory name:

```text
feat(c05-ingest-bridge): add reconnection logic to PgWriter
fix(c03-capture-pipeline): parameterize tshark fields per DLT type
docs(c02-database): add ERD for raw_packets hypertable
refactor(c09-rest-api): extract auth middleware
chore(skills): add github skill for SSH and commit safety
```

---

## Commit Body Rules

Every commit MUST include a body that describes:
1. **What** changed (bullet list for multi-file commits)
2. **Why** it changed (design rationale, bug reference, or decision link)

### Good Body

```text
Add reconnection logic to PgWriter with exponential backoff

What: PgWriter now retries failed COPY operations with delays of
1s, 2s, 4s, max 30s. A new `reconnect_attempts` counter is
incremented on each retry and reset on success.

Why: Database restarts during maintenance (systemd restart, package
upgrade) previously caused permanent ingest failure. The backoff
covers transient outages up to ~1 minute while avoiding thundering
herd on recovery. Per D-09, the 300K-row buffer absorbs the gap.
```

### Bad Body

```text
fixed stuff
```

```text
updated code
```

```text
wip
```

---

## Commit Granularity (Tian'er Rules)

- **One file per commit is the default.**
- **Bundle only when tightly coupled** — files that are part of the same logical change (e.g., a design doc and its associated ADR).
- **Never bundle unrelated changes.** A design document update and a skill file change go in separate commits unless they are the same logical unit.
- **When in doubt, split.**

### Decision Tree

| Files Changed | Commit Strategy |
|---------------|-----------------|
| 1 file | One commit with that file |
| 2-5 files, same component | One commit with scope: `docs(c03-capture-pipeline): ...` |
| 2-5 files, different components | Multiple commits, one per logical group |
| Design doc + associated ADR | One commit (tightly coupled) |
| Design doc + skill update | Two commits (different domains) |
| Multiple review files for same ticket | One commit: `docs(reviews): add retry-X specialist reviews` |
| New skill + AGENTS.md registry update | One commit (registry update is a consequence of the new skill) |

### Example: TIANER-001 Phase A Commits

```text
ccf3996 docs: add cross-document consistency rule and storage-strategy to registry    (1 file)
4b9106a docs: apply Phase A resolutions to inception and component-breakdown         (2 files, coupled)
bcbd91d docs: add container persistent storage strategy                              (1 file)
fea8eed docs: add ADR-0001 for all 24 Phase A decisions                             (1 file)
e9fc904 docs: add C01 platform infrastructure design document                       (1 file)
a377e19 docs: add C02 database design document                                      (1 file)
40c5969 docs: add retry-2, retry-3, and specialist expansion reviews                (multiple files, same domain)
23ca7a5 docs: add codified findings and resolutions for all 26 Phase A issues       (3 files, same domain)
3bcd84b chore: update pipeline passport TIANER-001 with Phase A completion          (1 file)
fa110c4 docs: update pipeline.md with Phase A gate results and specialist roster    (pipeline + skills)
```

---

## Commit Retry (Commitizen Pattern)

If a commit fails (e.g., pre-commit hook rejects it, tests fail after commit), do NOT create a new "fix typo" commit. Use the retry pattern:

```bash
# Amend the last commit with corrections
git add <fixed-files>
git commit --amend

# If the commit was already pushed, use --no-edit to keep the message
git commit --amend --no-edit
```

If tests fail post-commit: fix the code, stage changes, `git commit --amend`. The commit message stays correct — only the code changes.

---

## SSH Authentication (Tian'er Specific)

### The Agent Socket Problem

Subagent sessions do NOT inherit the user's `SSH_AUTH_SOCK` environment variable. Even when `ssh -T git@github.com` works in the user's terminal, a subagent sees `Permission denied (publickey)`. This is normal — the subagent runs in a different process context. This was discovered and diagnosed during the TIANER-001 Phase A commits.

### Solution

This project uses a persistent SSH agent socket at `~/.ssh/agent.sock`. All git push commands MUST prepend the explicit socket path:

```bash
SSH_AUTH_SOCK=~/.ssh/agent.sock git push origin main
```

### Pre-Flight Check (Before Any Push)

```bash
# Must return "successfully authenticated" before push is attempted
SSH_AUTH_SOCK=~/.ssh/agent.sock ssh -T git@github.com 2>&1 | grep -q "successfully authenticated" || {
    echo "SSH check failed — cannot push"
    exit 1
}
```

### Resolution Protocol

When a git push fails with `Permission denied (publickey)`, follow this sequence:

1. **Check for explicit agent socket:** `ls ~/.ssh/agent.sock` — if it exists, prepend `SSH_AUTH_SOCK=~/.ssh/agent.sock` to git commands
2. **Check SSH config:** `cat ~/.ssh/config` — verify `IdentityFile` is defined for `Host github.com`
3. **Verify with explicit key path:** `ssh -i ~/.ssh/github_key -T git@github.com`
4. **If all fail:** STOP. Report the key fingerprint to the user: `ssh-keygen -lf ~/.ssh/github_key.pub`
5. **Never attempt password-based auth.** This project uses SSH keys exclusively.

---

## Push Safety Rules

### Pre-Push Verification

Before pushing, verify:
1. **All commits follow conventional format:** `git log --oneline origin/main..HEAD` — check each message against the type table
2. **No WIP commits:** Search for "WIP", "tmp", "TODO commit", "fixup" in recent commit messages: `git log --oneline -10 | grep -iE 'wip|tmp|fixup|todo.commit'`
3. **Commits are logically grouped:** No single commit touches 20 unrelated files — `git diff --stat origin/main..HEAD` should show coherent groupings
4. **SSH is working:** Run the Pre-Flight Check

### Never Force Push to Main

`git push --force origin main` is FORBIDDEN. The `main` branch is the source of truth. Force-pushing rewrites history and destroys the audit trail of decisions, reviews, and design evolution.

If `main` and `origin/main` have diverged, use:
```bash
git pull --rebase origin main
git push origin main
```

### Push Commands

```bash
# Always with explicit SSH agent socket
SSH_AUTH_SOCK=~/.ssh/agent.sock git push origin main

# For feature branches
SSH_AUTH_SOCK=~/.ssh/agent.sock git push origin <branch-name>
```

### Post-Push Verification

```bash
# Confirm what was pushed
git log --oneline -5

# Verify remote matches local
git log --oneline origin/main -5
```

---

## After Every Push

The agent MUST update `README.md` if the push includes:
- New files created or deleted
- New components added
- Status changes (Phase A → Phase B, review pass → fail)

This is enforced by the `Documentation-Update Rule` in `AGENTS.md`.

---

## Troubleshooting Reference

| Symptom | Diagnosis | Fix |
|---------|-----------|-----|
| `Permission denied (publickey)` | Subagent doesn't inherit SSH_AUTH_SOCK | Use explicit socket: `SSH_AUTH_SOCK=~/.ssh/agent.sock` |
| `Could not read from remote repository` | Wrong remote URL or no network | Check `git remote -v`, verify network |
| `fatal: not a git repository` | Wrong directory | `cd` to repo root first |
| `Your branch is ahead of origin/main by N commits` | Unpushed commits exist | This is not an error — it means you have work to push |
| `merge conflict` | Local and remote diverged | NEVER force-push. `git pull --rebase origin main` |
| `error: failed to push some refs` | Remote has newer commits | `git pull --rebase` then retry push |
| Commit hook rejected | Commit message doesn't follow spec | Amend with `git commit --amend` with corrected message |

---

## Self-Reflection Clause

After any git-related failure, the agent MUST ask:
1. **Why did this failure occur?** — Was it an SSH issue, a commit format issue, or a grouping issue?
2. **What procedural check would have prevented it?** — Should we have run the Pre-Flight Check before attempting the operation?
3. **Update this skill** — Add the symptom and fix to the Troubleshooting Reference table so the same class of failure is caught earlier next time.
