---
name: pr-description
description: Generate a reviewer-ready PR description from the current branch diff and save it to pr_description.md. Enforces branch sync before generating.
user-invocable: true
---

# PR Description Generator

Produce an accurate, reviewer-ready PR description based strictly on the complete diff of the current branch against `origin/master`. Save the result to `pr_description.md` at the workspace root.

---

## When to Invoke

- When the user runs `/pr-description` and wants a ready-to-paste PR description
- When the user asks to "generate PR description", "write PR desc", or similar

---

## Agent Instructions

### Step 1: Enforce Branch Sync (BLOCKING)

Run the following to verify the local branch is fully synced with its remote:

```bash
git fetch origin
git status
git log --oneline HEAD..origin/$(git rev-parse --abbrev-ref HEAD) 2>/dev/null | head -20
```

**If `git status` shows "Your branch is behind" or if the log output is non-empty (missing commits from origin), STOP and return only:**

> The local branch is behind origin.
> Please sync your branch so that all commits from origin are present locally
> (e.g. run `git fetch origin` and then fast-forward your current branch),
> then regenerate the PR description.

Do NOT proceed if there is any indication that local is behind origin.

---

### Step 2: Collect Diff

Run:

```bash
git diff origin/master...HEAD --stat
git diff origin/master...HEAD -- ':!go.sum' ':!.claude/'
```

Parse:
- Total files changed, grouped by: added / modified / deleted
- Commit messages on the branch (`git log --oneline origin/master..HEAD`)
- Current branch name (for JIRA ticket reference extraction)

---

### Step 3: Generate PR Description

Base ALL content strictly on the diff. Never invent behaviour, intentions, or business logic not visible in the code.

Write the output in the following exact format:

---

## Summary

2–3 sentences covering:
- What the change does at a high level.
- Include JIRA ticket reference if present in the branch name or commit messages (e.g. `QC-27048`).
- High-level technical impact.

## What Changed

Group changes by category. Only include categories that have actual changes. Use bold headers:

**Backend** — implementation files (services, consumers, workers, handlers, etc.)
**API** — route definitions, request/response contracts
**Frontend** — UI components, views
**Tests** — new or modified test files; list each test case covered
**Config** — environment variables, `.env.example`, feature flags
**Infra** — Dockerfile, CI/CD, migrations, Helm charts
**Docs** — README, runbooks, swagger

Each bullet must reference a specific file path and describe the concrete change. No speculation.

## Testing

**Preconditions** — accounts, flags, env vars, seed data required

**Steps** — numbered, concrete, runnable

**Expected result** — what success looks like for each scenario

## Impact & Risks

- Affected systems and components
- Breaking changes or required migrations
- Performance or security implications
- Deployment considerations (e.g. env var must be set before rollout)
- Risk level: Low / Medium / High with brief justification

## Additional Notes

- Dependencies or required follow-up work
- Required config or environment changes not covered in Testing
- Deployment order or feature flag sequencing

## Follow-ups

Actionable items that are out of scope for this PR but should be tracked:
- Unanswered questions a reviewer or author should address
- Runbook or documentation gaps
- Potential follow-on PRs
- Anything the diff does not make fully clear (e.g. external dependencies, downstream consumers)

---

### Step 4: Save Output

Write the generated description to `pr_description.md` at the workspace root. Overwrite if it already exists.

---

## Style Rules

- Use precise technical language: file paths, function names, constant names, env var names.
- Do not use emojis in section headers.
- Wrap symbol names in backticks: `CreditLevelMonitorJob`, `GCHAT_CREDIT_LEVEL_WEBHOOK_URL`.
- Keep bullets scannable — one concrete fact per bullet.
- Do not add a section that has no content.
- Do not speculate about intent; only describe what the diff shows.
