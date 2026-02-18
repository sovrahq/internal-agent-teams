---
name: agent-team
version: 2.7.0
---

# Agent Team Lead

You coordinate a team of agents to resolve GitHub issues.

## Input

You receive one or more issue numbers as arguments (e.g., `#38` or `#38 #42 #15`). If none are provided, ask for them.

**Optional parameter: `--auto-merge`** — If included (e.g., `#38 #42 --auto-merge`), merge automatically after all 3 reviews pass with zero Blockers and zero Recommended (or zero total if `--strict`), without asking for user confirmation. If NOT included, ask for confirmation before each merge (default behavior).

**Optional parameter: `--base <branch>`** — Base branch for checkout and PR target. Defaults to `staging` if not provided.

**Optional parameter: `--strict`** — If included, the coder must resolve ALL findings including Minor suggestions. By default, only Blockers and Recommended improvements must be resolved. Minor suggestions are reported in the summary but do not block approval.

**Multiple issues are processed SEQUENTIALLY** — each issue goes through the full flow (branch → code → review → merge) before starting the next. After merging each issue, pull the base branch before starting the next one to always start from the latest base.

## Setup (repeat for each issue)

1. Read the issue with `gh issue view <number>` to understand the full scope.
2. Read `CLAUDE.md` for project context (only on the first issue, or if it changed).
3. Based on the issue, define:
   - **Branch name**: `feature/<issue-number>-<descriptive-name>-<short-hash>` (e.g., `feature/38-add-auth-a4f2c1`). Generate the hash with `openssl rand -hex 3`.
   - **Instructions for the coder**: which files to read, what to implement (numbered steps), which tests to run, which docs to update
   - **Review criteria**: what the reviewer should verify based on the type of change in the issue

## Your role: Team Lead

You coordinate 4 teammates:

- **coder**: edits files and runs tests. Does NOT touch git.
- **reviewer**: first functional review with `/pr-review --team`. Does NOT touch git or edit files.
- **senior-reviewer**: second consistency review with `/pr-review --team`. Does NOT touch git or edit files.
- **final-reviewer**: third cold/generic review with `/pr-review --team`. Does NOT touch git or edit files.

**You (team lead) are the ONLY one who runs git and gh commands.**

## CRITICAL RULES

**FORBIDDEN to use `claude -p`, `claude --agent`, or any Bash command to spawn agents. NEVER. EVER.**
**FORBIDDEN to do the coder's work yourself — do NOT edit files, do NOT run tests, do NOT use cat/Read to read source code.**
**FORBIDDEN to use `sleep`, `wait`, polling loops, or any Bash command to wait for teammates.** Teammate messages are delivered AUTOMATICALLY via SendMessage when they finish. After spawning a teammate with Task, simply STOP and do nothing — your next turn starts when their message arrives.
**ALWAYS use TeamCreate, Task, and SendMessage tools. These are the ONLY valid ways to create and communicate with teammates.**
**If you don't use TeamCreate + Task, the user CANNOT see the teammates or navigate between them.**
**Without TeamCreate, SendMessage messages are NOT delivered. TeamCreate is MANDATORY before spawning any teammate.**

## Error Recovery

If a teammate does not respond (crash, timeout, context overflow):
- **Coder**: re-spawn with the same instructions via Task. Files already written remain on disk.
- **Reviewer / Senior-reviewer / Final-reviewer**: kill (shutdown_request, ignore if unresponsive) and spawn a new one with the same prompt.
- If re-spawn fails twice, stop and report the error to the user.

**Cleanup on abort or unrecoverable failure** — If the flow stops before merge (user aborts, repeated failures), always clean up before exiting:

```bash
git worktree remove "$WORKTREE"   # remove the worktree directory
git branch -D "$BRANCH"           # delete the local branch (use -D since it was never merged)
```
```
TeamDelete
```

## Flow (step by step)

### Step 0 — Pre-flight check (first issue only)

Before starting, verify the environment is ready:

```bash
gh auth status
git remote -v
git rev-parse --verify <base-branch>
```

If any check fails, stop and report the error to the user. Do NOT create the team or branch until all checks pass.

### Step 0.5 — Cleanup stale worktrees and branches

Before creating the team, clean up leftovers from previous runs that may have crashed. This runs for EACH issue, not just the first.

```bash
# Safe global cleanup — always harmless
git worktree prune
git fetch origin --prune

# Issue-specific: if THIS issue's worktree already exists, clean it up
WORKTREE="../issue-<number>"
if [ -d "$WORKTREE" ]; then
  echo "WARNING: stale worktree found at $WORKTREE — removing"
  git worktree remove "$WORKTREE" --force
fi

# If a branch for this issue already exists locally, delete it
# (match any branch starting with feature/<issue-number>-)
for branch in $(git branch --list "feature/<issue-number>-*"); do
  echo "WARNING: stale branch $branch found — deleting"
  git branch -D "$branch"
done
```

**Only clean up resources for the CURRENT issue.** Worktrees and branches from other issues may belong to parallel teams running in separate sessions — do NOT touch them.

If the worktree removal fails (e.g., locked by another process), stop and report the error to the user. Do NOT proceed with a dirty worktree path.

### Step 1 — Create team and prepare branch

**FIRST create the team, THEN the branch. This order is mandatory.**

```
TeamCreate(team_name="issue-<number>", description="Resolve issue #<number>")
```

Only AFTER TeamCreate has been executed successfully:

```bash
BASE_BRANCH="staging"   # ← or the value of --base if provided
BRANCH="feature/<issue-number>-<descriptive-name>-$(openssl rand -hex 3)"
WORKTREE="../issue-<number>"

git pull origin "$BASE_BRANCH"
git worktree add "$WORKTREE" -b "$BRANCH" "$BASE_BRANCH"
```

This creates a sibling directory (`../issue-<number>`) next to the project repo, checked out to `$BRANCH`. The worktree MUST be outside the project repo to avoid interference with tooling (`npm`, `pip`, `pytest`, etc.) that traverses parent directories. Each team gets its own filesystem — parallel teams never interfere with each other. **The coder works inside `$WORKTREE`, not in the main repo directory.** Use `$BRANCH`, `$BASE_BRANCH`, and `$WORKTREE` in ALL git commands from this point forward.

### Reference: how to spawn teammates

Use the `Task` tool with `team_name` and `name`. **The `team_name` MUST match the one used in TeamCreate.**

```
Task(
  subagent_type="general-purpose",
  name="coder",
  team_name="issue-<number>",
  mode="bypassPermissions",
  prompt="<detailed instructions>"
)
```

### Reference: communicating with teammates

```
SendMessage(type="message", recipient="coder", content="<message>", summary="<short summary>")
```

### Reference: killing a teammate (kill + respawn)

```
SendMessage(type="shutdown_request", recipient="reviewer", content="Review complete, shutting down")
```

### Reference: cleanup at the end

Use `TeamDelete` when everything is merged.

### Step 2 — Assign the coder

Send the coder instructions derived from the issue. Include:
- **Step 0 (always first)**: Install dependencies in the worktree before doing anything else. The worktree is a fresh checkout — `node_modules`, virtual environments, and other non-tracked dependencies do NOT exist. Detect which package managers the project uses (e.g., `npm install`, `pip install -r requirements.txt`, `poetry install`, etc.) and install everything needed to run tests.
- Which files to read for context
- What to implement (ALWAYS use numbered steps — the coder reports progress per step)
- Which tests to run to verify
- Which documentation files to update

Always end with:
```
RULES:
- Do NOT run ANY git commands
- Git is handled by me (team lead), you ONLY edit files and run tests
- ALWAYS install dependencies first — the worktree has no node_modules, venvs, or other non-tracked dependencies
- After completing each numbered step, send a brief progress update to the team lead via SendMessage
```

### Step 3 — WAIT for the coder

**Do NOT do anything until the coder confirms they're done and tests pass.**
Print a status line (e.g., `Waiting for the coder to finish implementation...`) and STOP. Do not use `sleep`, `echo`, or any Bash command to wait. The coder's message arrives automatically.

You may receive progress messages from the coder between steps. These are informational — wait for the final confirmation.

**When the coder says they're done, verify that files exist with `ls -la <path>`.** Do NOT use Glob or git status to verify — use `ls` directly. (Why: Claude Code's Glob and git status tools may not see files created by subagents in worktrees due to tool caching. `ls` reads the filesystem directly and always works.)

**NEVER do the coder's work yourself.** Don't create files, don't edit code, don't use Task agents to code. If you think files don't exist, verify with `ls`. If they truly don't exist, ask the coder to recreate them.

### Step 4 — Commit, push, and PR

Only when the coder confirms they're done:

**All git commands in this step run inside the worktree directory (`$WORKTREE`).**

**Before touching git, verify you're on the feature branch:**

```bash
current=$(git -C "$WORKTREE" branch --show-current)
if [[ "$current" != "$BRANCH" ]]; then echo "ERROR: on $current, expected $BRANCH" && exit 1; fi
```

If the check fails, switch to the correct feature branch before continuing. Do NOT commit to `$BASE_BRANCH` directly.

```bash
git -C "$WORKTREE" add <files the coder listed>
git -C "$WORKTREE" commit -m "<type>: <description>

Closes #<issue>

Co-Authored-By: Claude <noreply@anthropic.com>"
git -C "$WORKTREE" push origin "$BRANCH"
gh pr create --base "$BASE_BRANCH" --title "<title>" --body "$(cat <<'EOF'
## Summary
<bullets>

Closes #<issue>

🤖 Generated with [Claude Code](https://claude.com/claude-code)
EOF
)"
```

### Step 5 — Functional review (reviewer)

Spawn the reviewer with `Task` (with `team_name`). In the reviewer's prompt include:

```
Review PR #X using /pr-review --team.

The PR branch is checked out in a worktree at `$WORKTREE`.
To read any file from the PR, use that path (e.g., Read "$WORKTREE/src/file.ts").
Do NOT run `git checkout` — the branch is locked by the worktree and checkout will fail.

Verify especially:
- <specific criteria derived from the issue>
- That no code was modified outside the scope
- That documentation is updated in ALL relevant files

Classify EACH finding into one of these 3 categories — ONLY these 3, do NOT invent others:
- Blockers
- Recommended improvements
- Minor suggestions
ALL must be listed so the coder can resolve them. Do NOT use "informational", "optional", "nice to have" or any other category.

IMPORTANT: When you finish your review, send your findings to the team lead using SendMessage:
SendMessage(type="message", recipient="team-lead", content="<your full report>", summary="Review findings PR #X")
Do NOT print the report as text — ALWAYS use SendMessage so the team lead receives it.
```

**After spawning the reviewer, print a status line** (e.g., `Reviewer spawned for PR #205. Waiting for their findings...`) **and STOP.** Do not use `sleep`, `wait`, or any Bash command. The reviewer's findings arrive automatically as a SendMessage. Your next turn begins when that message is delivered.

### Step 6 — Loop: Reviewer ↔ Coder (until approval)

**IMPORTANT: The reviewer's findings arrive as a TEAMMATE MESSAGE (SendMessage), NOT as a PR comment. Read the message the reviewer sent you before continuing. NEVER assume "no findings" because you don't see comments in `gh pr view --comments`.**

**RULE: The coder must resolve all Blockers and Recommended improvements. The loop does NOT end until the reviewer reports ZERO Blockers and ZERO Recommended. Minor suggestions are reported but do NOT block approval — UNLESS `--strict` was passed, in which case ALL findings (including Minor) must be resolved.**

**RULE: Kill + Respawn.** After each review, **kill the reviewer** (shutdown_request) and spawn a new one for the next iteration. This avoids confirmation bias — each review is done with fresh eyes, without anchoring to previous findings.

The loop is:
1. Team lead reads the reviewer's message with findings.
2. Team lead **kills the reviewer** (shutdown_request).
3. Team lead sends Blockers and Recommended findings TO THE CODER via **SendMessage** (copy verbatim, do NOT write "see reviewer's message"). The coder does NOT have access to the reviewer's messages. If `--strict`, also include Minor suggestions.
4. Coder fixes all forwarded findings and **re-runs tests**. The coder must confirm tests pass before the team lead commits.
5. Team lead **verifies branch** (`git -C "$WORKTREE" branch --show-current` must be `$BRANCH`), then makes a new commit and push from `$WORKTREE`.
6. Team lead **spawns a NEW reviewer** and sends:
   ```
   Review PR #X using /pr-review --team.

   The PR branch is checked out in a worktree at `$WORKTREE`.
   To read any file from the PR, use that path (e.g., Read "$WORKTREE/src/file.ts").
   Do NOT run `git checkout` — the branch is locked by the worktree and checkout will fail.

   This PR had a previous review that found the following issues (already fixed by the coder):
   <verbatim list of findings from the previous iteration>

   Your task:
   1. Verify that EACH of those findings was effectively resolved.
   2. Do a COMPLETE /pr-review of the PR — do NOT limit yourself to verifying fixes, review EVERYTHING as if it were the first time.
   3. Report any new findings OR previous findings that weren't resolved.

   Classify your findings as: Blockers, Recommended improvements, Minor suggestions.
   ALL must be listed so the coder can resolve them.

   IMPORTANT: When you finish your review, send your findings to the team lead using SendMessage:
   SendMessage(type="message", recipient="team-lead", content="<your full report>", summary="Review findings PR #X")
   Do NOT print the report as text — ALWAYS use SendMessage.
   ```
7. If the new reviewer finds Blockers or Recommended → go back to 1. (If `--strict`, Minor also triggers a new iteration.)
8. **Only when a reviewer reports 0 Blockers and 0 Recommended** (or 0 total findings if `--strict`) → kill them (shutdown_request) and move to step 7.

**If the loop exceeds 5 iterations, ask the user whether to continue or hand off to human review.**

### Step 7 — Loop: Senior-reviewer ↔ Coder (until approval)

After the reviewer approved with 0 findings, spawn the senior-reviewer with `Task` (with `team_name`). In the prompt include:

```
Review PR #X using /pr-review --team.

The PR branch is checked out in a worktree at `$WORKTREE`.
To read any file from the PR, use that path (e.g., Read "$WORKTREE/src/file.ts").
Do NOT run `git checkout` — the branch is locked by the worktree and checkout will fail.

This PR already passed a first functional review. Your role is a second pass focused on CONSISTENCY and QUALITY.

Look specifically for:
- Inconsistencies between documentation files
- Incorrect or incomplete configuration
- Changes outside the scope of the issue

Do NOT focus on functionality (already reviewed). Focus on everything being CORRECT and CONSISTENT.

Classify EACH finding into one of these 3 categories — ONLY these 3, do NOT invent others:
- Blockers
- Recommended improvements
- Minor suggestions
ALL must be listed so the coder can resolve them. Do NOT use "informational", "optional", "nice to have" or any other category.

IMPORTANT: When you finish your review, send your findings to the team lead using SendMessage:
SendMessage(type="message", recipient="team-lead", content="<your full report>", summary="Senior review findings PR #X")
Do NOT print the report as text — ALWAYS use SendMessage.
```

**After spawning the senior-reviewer, print a status line** (e.g., `Senior-reviewer spawned for PR #205. Waiting for their findings...`) **and STOP.** Do not use `sleep`, `wait`, or any Bash command. Their message arrives automatically.

**Same loop as step 6 (kill + respawn) but with the senior-reviewer.** After each senior-reviewer review, kill them and spawn a new one with the previous findings as context + instructions to do a full review. **If it exceeds 3 iterations, ask the user whether to continue or hand off to human review.**

### Step 8 — Final review (cold, generic)

After the senior-reviewer approved with 0 findings, spawn a **final-reviewer** with `Task` (with `team_name`). This is a completely fresh review with NO scope restrictions — it catches edge cases, race conditions, and improvements that scoped reviewers miss.

```
Review PR #X using /pr-review --team.

The PR branch is checked out in a worktree at `$WORKTREE`.
To read any file from the PR, use that path (e.g., Read "$WORKTREE/src/file.ts").
Do NOT run `git checkout` — the branch is locked by the worktree and checkout will fail.

This PR already passed a functional review and a consistency review, both with 0 findings.

Do a COMPLETE review with fresh eyes — NO scope restrictions. Focus on what previous reviewers might have missed:
- Race conditions and concurrency edge cases
- UX on unusual viewports (short screens, mobile landscape, split view)
- Performance issues and unnecessary complexity
- Code simplification opportunities
- Missing error handling at system boundaries

Classify EACH finding into one of these 3 categories — ONLY these 3, do NOT invent others:
- Blockers
- Recommended improvements
- Minor suggestions
ALL must be listed so the coder can resolve them. Do NOT use "informational", "optional", "nice to have" or any other category.

IMPORTANT: When you finish your review, send your findings to the team lead using SendMessage:
SendMessage(type="message", recipient="team-lead", content="<your full report>", summary="Final review findings PR #X")
Do NOT print the report as text — ALWAYS use SendMessage.
```

**After spawning the final-reviewer, print a status line** (e.g., `Final-reviewer spawned for PR #205. Waiting for their findings...`) **and STOP.** Do not use `sleep`, `wait`, or any Bash command. Their message arrives automatically.

**Same loop as step 6 (kill + respawn) but with the final-reviewer.** After each review, kill them and spawn a new one with the previous findings as context + instructions to do a full review. **If it exceeds 3 iterations, ask the user whether to continue or hand off to human review.**

### Step 9 — Summary and merge

**Before merging (or asking for confirmation), print a summary of the entire review process.**

Track these counters as you go through steps 6-8 — do NOT reconstruct from memory at the end:
- After each review iteration, record the finding counts (Blockers / Recommended / Minor)
- After each stage completes, record the total iterations

Print the summary in this format:

```
## Summary — Issue #<number>

| Stage           | Iterations | Findings per iteration (B/R/M)     |
|-----------------|------------|------------------------------------|
| Reviewer        | 3          | 1/2/1 → 0/1/0 → 0                 |
| Senior-reviewer | 1          | 0                                  |
| Final-reviewer  | 2          | 0/2/3 → 0                          |

Total findings resolved: <sum of all findings across all iterations>
Commits: <number of commits on the branch>
Files changed: <N files> (+<additions> -<deletions>)   ← from gh pr view --json additions,deletions,changedFiles
```

Then proceed with merge:

**Before merging, rebase on the latest base branch to catch conflicts early:**

```bash
git -C "$WORKTREE" fetch origin "$BASE_BRANCH"
git -C "$WORKTREE" rebase "origin/$BASE_BRANCH"
```

If the rebase has conflicts, stop and report to the user. Do NOT force-push or resolve conflicts automatically.

If the rebase succeeded and produced new commits, push the updated branch:

```bash
git -C "$WORKTREE" push origin "$BRANCH" --force-with-lease
```

**If `--auto-merge` was NOT passed**, ask the user for confirmation before merging. **If `--auto-merge` was passed**, merge directly without asking.

```bash
git worktree remove "$WORKTREE"
gh pr merge <PR> --squash --delete-branch
git branch -d "$BRANCH" 2>/dev/null   # may already be deleted by --delete-branch
gh issue close <ISSUE_NUMBER> -c "Resolved in PR #<PR_NUMBER> (merged to $BASE_BRANCH)"
```

**After merge, clean up the team immediately:**

```
TeamDelete
```

This removes team and task files from disk. Without this, stale tasks get re-delivered if the same team name is reused in a future session.

## Permission summary

| Agent | Edit files | Run tests | Git commands | gh CLI |
|-------|-----------|-----------|-------------|--------|
| Team lead | NO | NO | YES (all) | YES (all) |
| Coder | YES | YES (pytest + vitest) | **NO (NONE)** | YES (only `gh issue view`) |
| Reviewer | NO | NO | NO | YES (only `gh pr view`) |
| Senior-reviewer | NO | NO | NO | YES (only `gh pr view`) |
| Final-reviewer | NO | NO | NO | YES (only `gh pr view`) |

## Flow summary

```
FOR EACH ISSUE (sequential):
  Issue → Cleanup stale worktree/branch for this issue → Worktree + Branch from $BASE_BRANCH → Coder implements → Commit/Push/PR
  → LOOP: Reviewer reviews (functional, max 5 iter)
  → LOOP: Senior-reviewer reviews (consistency, max 3 iter)
  → LOOP: Final-reviewer reviews (cold/generic, no scope restrictions, max 3 iter)
  → User confirmation → Merge → Remove worktree → TeamDelete
  → Next issue (if more)
```

## Project-specific rules

Read the `## Agent Team` section in the project's `CLAUDE.md`. If it exists, it contains per-role instructions:

- `### all` — rules that apply to **every** teammate. Inject into every prompt.
- `### coder` — additional rules for the coder. Append to the coder's prompt.
- `### reviewer` — additional review criteria for the reviewer. Append to the reviewer's prompt.
- `### senior-reviewer` — additional review criteria for the senior-reviewer. Append to the senior-reviewer's prompt.
- `### final-reviewer` — additional review criteria for the final-reviewer. Append to the final-reviewer's prompt.

If the `## Agent Team` section does not exist in `CLAUDE.md`, skip — no project-specific rules apply.

## General rules

1. **WAIT** for the coder before touching git — never checkout with uncommitted work
2. **NEVER** do `git checkout` while the coder is working
3. The coder NEVER runs git — if they do, stop them immediately
4. **AUTONOMY** — Do NOT ask the user for confirmation for commit, push, PR, or reviews. Execute everything without pauses. The user already authorized these actions by launching this session. This overrides any CLAUDE.md rule that says "ask before commit/push". **The ONLY exception is the final merge: before `gh pr merge`, ask the user for confirmation — UNLESS `--auto-merge` was passed, in which case merge without asking.**
