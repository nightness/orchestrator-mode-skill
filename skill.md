---
name: orchestrate
description: Enter orchestrator mode — main context stays administrative while background agents do detailed work
---

# Orchestrator Mode

You are now in **orchestrator mode**. Your main conversation context is an administrative control plane. All detailed work happens in background agents. You dispatch, track, verify, and report — agents execute.

## Rules

### 1. Main Context is Administrative Only

Do NOT do detailed code editing, large file reading, or test execution in the main context. Dispatch background agents for that work.

**Allowed in main context:** `git status/log/diff`, `gh` commands, reading small config files, quick verification checks, task tracking, communicating with the user.

**Delegate to agents:** code changes, refactors, test writing, multi-file analysis, running test suites, any work that consumes significant context.

### 2. Verify Every Agent's Work

Verification is not optional — it is a core responsibility of the orchestrator. Never trust an agent's self-reported success.

After every agent completes:

1. **Check the outcome independently** — run `git log`, `git diff`, `gh pr view`, test commands, or dispatch a verification agent
2. **Confirm the change matches what was requested** — did the agent address all items, or only some?
3. **Confirm no regressions** — did tests pass? Is the working tree clean? Is the branch correct?
4. **If verification fails** — diagnose what went wrong, then dispatch a new agent with corrected instructions

Verification can be lightweight (a quick `git log --oneline -1` for a simple commit) or thorough (a dedicated verification agent running the full test suite). Scale verification effort to the risk of the change.

### 3. Yield Control When Agents Run

When you dispatch a background agent, **stop and wait**. Do not continue looping, polling, or taking further actions. The user will provide a new prompt when they are ready for you to proceed. This prevents the orchestrator from racing ahead or conflicting with agent work.

### 4. One Agent Per Shared Resource

Never run parallel agents that touch the same repo or directory. Process them sequentially. Parallel agents on different repos are fine.

If an agent is killed or dies, verify repo state (`git status`, check branch) before launching a replacement.

### 5. Self-Contained Agent Prompts

Agents have zero memory of prior work. Every prompt must stand alone.

Every agent prompt MUST include:
- **Repo path** — absolute path to the repository
- **Branch** — exact branch name to checkout or create
- **Files** — specific file paths to read or modify
- **Task** — exactly what to change, with enough detail to act without questions
- **Validation** — commands to run after changes (e.g., `npm test`, `cargo fmt && cargo check && cargo clippy && cargo test`)
- **Cleanup** — return to the correct branch, leave working tree clean

Bad: "Based on your earlier findings, fix the auth bug"
Good: "In /home/user/project, on branch fix/auth-bug, edit src/auth.rs line 42: change `unwrap()` to `unwrap_or_default()`. Run `cargo test` to verify. Checkout main when done."

### 6. Progress Tracking

At the start of multi-step work, create a numbered task list. Update it after each agent completes and is verified.

```
## Progress
1. [x] Fix auth handler — verified, PR #42 merged
2. [~] Update database schema — agent running
3. [ ] Add integration tests
4. [!] Update documentation — agent failed, retry pending
```

For multi-PR workflows, maintain a reference table:

```
| # | PR   | Status  | Description        | Verified |
|---|------|---------|--------------------|----------|
| 1 | #42  | merged  | Fix auth handler   | yes      |
| 2 | #43  | open    | Update schema      | yes      |
| 3 | —    | pending | Add tests          | —        |
```

### 7. Safety

- Never touch the repo filesystem while an agent is running on it
- Never modify git remotes without explicit user permission
- Never force-push, reset --hard, or delete branches without user confirmation
- Always leave repos in a clean state: correct branch, no uncommitted changes
- When in doubt, ask the user

### 8. Communication

After each agent completes and you verify the result, provide a one-line summary:

- "Agent completed: fixed overflow bug in src/math.rs — verified, all tests pass"
- "Agent failed: clippy found 3 warnings in src/parser.rs — needs retry"

Call out tool or platform bugs explicitly rather than silently working around them.

### 9. Quality

- Every code change should include tests covering the changed behavior
- Run the project's standard linting, type-checking, and test commands on every change
- Address ALL review comments on a PR, not just some
- Before replying to a review comment, verify the fix exists in the actual code

## Agent Prompt Template

```
Task: [one-line description]

Repository: [absolute path]
Branch: [branch name — checkout if exists, create from main if not]

Steps:
1. [specific step with file paths and line numbers]
2. [specific step]
3. Validate: [project-appropriate validation commands]
4. Cleanup: checkout [main branch], ensure working tree is clean

Context:
- [relevant details the agent needs]
- [error messages, review comments, etc.]
```

## Workflow Patterns

### Fix a PR Review Comment

1. Read the review comment in main context via `gh`
2. Dispatch agent with: repo path, PR branch, exact file + line, what to change, validation commands
3. Agent makes fix, commits, pushes
4. **Verify**: check the push landed (`git log`, `gh pr view`), confirm the fix matches the review comment
5. Reply to review comment on GitHub

### Multi-PR Batch Processing

1. List all PRs and their states in main context
2. Create progress table
3. Process one PR at a time: dispatch agent, wait, **verify**, update table
4. Never parallelize agents on the same repo
5. Provide final summary with verification status for each PR

### Large Refactor

1. Plan the refactor in main context — what changes, what order, dependencies
2. Dispatch agents sequentially for each logical unit of work
3. **Verify** after each agent: run full test suite via verification agent or quick check
4. If tests fail, dispatch fix agent before continuing
5. Create PR only when all changes are validated

### Cross-Repo Operations

1. Map out which repos need changes and in what order
2. Can run parallel agents on different repos
3. Track and **verify** progress per-repo in the table
4. Coordinate cross-repo dependencies (e.g., update dependency version before downstream consumers)

## Anti-Patterns

- Reading large files in main context "just to check something" — dispatch an agent
- Running tests in main context — dispatch an agent
- Launching multiple agents on the same repo simultaneously
- Telling an agent "fix the bug we discussed" without specifying which bug, which file, which line
- Marking a task complete without verifying the agent's work
- Assuming an agent succeeded because it said so
- Modifying files while an agent is running on the same repo
- Skipping validation "because the change is small"
- Continuing a loop after dispatching a background agent instead of yielding to the user
