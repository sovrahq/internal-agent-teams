# Agent Teams

> Reusable agent team workflow for coordinating Claude Code sessions across projects. One command spawns a team lead + coder + 3 reviewers that resolve GitHub issues end-to-end.

## What it does

Turns a GitHub issue number into a merged PR — autonomously. The main Claude Code session reads `agent-team.md` and becomes the **team lead**, using `TeamCreate`, `Task`, and `SendMessage` to spawn and coordinate 4 specialized teammates. Each issue goes through: branch creation, implementation, 3 review stages (functional → consistency → cold/generic), and squash merge.

The team lead never edits files or runs tests. The coder never touches git. Reviewers never edit code. This strict permission model prevents agents from stepping on each other.

## Setup

Clone this repo alongside your projects:

```bash
git clone git@github.com:sovrahq/agent-teams.git
```

Add this to your project's Claude Code memory (MEMORY.md or CLAUDE.md), adjusting the path:

```
## Agent Team (swarm)
When the user says "agent-team <issues>", read `<path-to>/agent-teams/agent-team.md` and execute the flow described there.
Flow: team lead + coder + reviewer + senior-reviewer + final-reviewer.
Invoked with issue numbers as arguments (e.g., `agent-team #<issue-number>` or `agent-team #<issue> #<issue> --auto-merge`).
```

## Usage

```
agent-team #38                             # resolve a single issue
agent-team #38 #42 #15                     # resolve multiple issues sequentially
agent-team #38 --auto-merge                # merge without asking for confirmation
agent-team #38 --base main                 # use main instead of staging as base branch
agent-team #38 #42 --auto-merge --base main  # combine flags
```

### Parameters

| Parameter | Description | Default |
|-----------|-------------|---------|
| `#<number>` | GitHub issue numbers to resolve (1 or more, processed sequentially) | Required |
| `--auto-merge` | Merge automatically after all 3 reviews pass with zero findings | Ask for confirmation |
| `--base <branch>` | Base branch for checkout and PR target | `staging` |

## What you get

- **Autonomous issue resolution** — from issue to merged PR without manual intervention
- **3-stage review pipeline** — functional, consistency, and cold/generic review with kill + respawn
- **Strict permission model** — each agent can only do what it's supposed to do
- **Progress visibility** — coder reports per-step progress, reviewers report findings via SendMessage
- **Review summary** — iteration counts, finding counts, and total resolved before merge
- **Error recovery** — crashed teammates are automatically re-spawned (max 2 retries)
- **Project-specific rules** — per-role instructions via `CLAUDE.md` sections
- **Pre-flight checks** — verifies `gh auth`, git remote, and base branch before starting
- **Parallel-safe** — git worktrees isolate each team's filesystem, no branch collisions

## Examples

### Session start

The team lead runs pre-flight checks, creates the team, and spawns the coder:

```
✓ Pre-flight: gh auth OK, remote origin OK, staging exists

TeamCreate(team_name="issue-38", description="Resolve issue #38")

Worktree: ../issue-38 → feature/38-add-user-preferences-a4f2c1
Coder spawned with 5 implementation steps. Waiting for the coder to finish implementation...
```

### Coder progress updates

The coder reports progress per numbered step:

```
[coder → team-lead]: Step 1/5 complete — created PreferencesService with CRUD methods
[coder → team-lead]: Step 2/5 complete — added API routes and validation
[coder → team-lead]: Step 3/5 complete — migration file created
[coder → team-lead]: Step 4/5 complete — 12 tests written, all passing
[coder → team-lead]: Step 5/5 complete — updated API docs. All done, 12/12 tests passing.
```

### Review loop

Each reviewer examines the PR and reports findings. The team lead forwards them to the coder:

```
[reviewer → team-lead]: Review findings PR #205

## Blockers
- B1: SQL injection in sort parameter (preferences.py:47)

## Recommended improvements
- R1: Missing composite index for new query pattern
- R2: Error response doesn't include field name

## Minor suggestions
- M1: `get_prefs` → `get_preferences` for consistency

Verdict: REQUEST CHANGES — 1 blocker, 2 recommended, 1 minor
```

```
→ Killed reviewer. Forwarding 4 findings to coder...
→ Coder fixing all findings...
→ New commit pushed. Spawning fresh reviewer for iteration 2...
```

### Review summary before merge

After all 3 review stages pass, the team lead prints a summary:

```
## Summary — Issue #38

| Stage           | Iterations | Findings per iteration (B/R/M)     |
|-----------------|------------|------------------------------------|
| Reviewer        | 3          | 1/2/1 → 0/1/0 → 0                 |
| Senior-reviewer | 1          | 0                                  |
| Final-reviewer  | 2          | 0/2/3 → 0                          |

Total findings resolved: 10
Commits: 4
Files changed: 8 (+342 -27)
```

### Multiple issues

When multiple issues are provided, each goes through the full pipeline before starting the next:

```
✓ Issue #38 merged. Worktree ../issue-38 removed.

Starting issue #42...

TeamCreate(team_name="issue-42", description="Resolve issue #42")
Worktree: ../issue-42 → feature/42-fix-webhook-auth-e7b3d9
...
```

## How it works

### Agents and permissions

| Agent | Role | Edit files | Run tests | Git/gh |
|-------|------|-----------|-----------|--------|
| **Team lead** | Coordination, git, PRs | NO | NO | YES (all) |
| **Coder** | Implementation | YES | YES | NO (only `gh issue view`) |
| **Reviewer** | Functional review (`/pr-review --team`) | NO | NO | NO (only `gh pr view`) |
| **Senior reviewer** | Consistency review (`/pr-review --team`) | NO | NO | NO (only `gh pr view`) |
| **Final reviewer** | Cold generic review (`/pr-review --team`) | NO | NO | NO (only `gh pr view`) |

### Flow

```
PER ISSUE (sequential):
  0. Pre-flight check (gh auth, git remote, base branch)
  1. Team lead reads issue, creates worktree + branch from $BASE_BRANCH
  2. Team lead spawns coder with detailed numbered steps
  3. Coder implements, reports progress per step, runs tests
  4. Team lead commits, pushes, creates PR to $BASE_BRANCH
  5. Reviewer loop — functional review (max 5 iterations)
  6. Senior-reviewer loop — consistency review (max 3 iterations)
  7. Final-reviewer loop — cold/generic review (max 3 iterations)
  8. Summary → user confirms → squash merge → remove worktree → TeamDelete
  9. Next issue (if any)
```

### Review loops

The review process uses a **kill + respawn** pattern to avoid confirmation bias:

1. Reviewer examines the PR using `/pr-review --team` and reports findings via SendMessage
2. Team lead **kills the reviewer** (shutdown) and forwards findings to the coder
3. Coder fixes **all** findings (blockers, improvements, AND minor suggestions)
4. Team lead commits, pushes, and **spawns a fresh reviewer** with:
   - The previous findings as context (to verify they were resolved)
   - Instructions to do a full review, not just verify fixes
5. Loop continues until a reviewer approves with **zero findings**

**Iteration limits:**

| Stage | Max iterations | Escalation |
|-------|---------------|------------|
| Reviewer (functional) | 5 | Ask user: continue or hand off to human review |
| Senior reviewer (consistency) | 3 | Same |
| Final reviewer (cold/generic) | 3 | Same |

### Error recovery

If a teammate crashes, times out, or overflows context:

| Agent | Recovery |
|-------|----------|
| Coder | Re-spawn with same instructions. Files on disk are preserved. |
| Reviewer / Senior / Final | Kill (shutdown_request) and spawn a new one with same prompt. |

If re-spawn fails twice, the team lead stops and reports the error to the user.

### Merge behavior

- **Default**: asks user for confirmation before `gh pr merge --squash`
- **`--auto-merge`**: merges automatically after all 3 reviews pass with zero findings

## Project-specific rules

The team lead reads the `## Agent Team` section in your project's `CLAUDE.md` and injects per-role instructions into each teammate's prompt. This keeps `agent-team.md` generic while letting each project define its own rules.

### Convention

Add this section to your project's `CLAUDE.md`:

```markdown
## Agent Team

### all
<!-- Rules injected into EVERY teammate's prompt -->
- If the change resolves a finding in `docs/audit.md`, mark it with ~~strikethrough~~ and `RESOLVED`
- Update the `Last updated` field in the audit report header

### coder
<!-- Additional rules for the coder only -->
- Run `scripts/post-merge.sh` is NOT manual — only runs post-merge by CI

### reviewer
<!-- Additional review criteria for the reviewer -->
- Verify audit report findings are marked as resolved when applicable
- Verify `Last updated` field is updated with date and context

### senior-reviewer
<!-- Additional review criteria for the senior-reviewer -->
- Verify consistency between audit report status and actual code changes

### final-reviewer
<!-- Additional review criteria for the final-reviewer -->
- Check for edge cases in error handling at system boundaries
```

### How it works

1. Team lead reads `CLAUDE.md` during setup
2. If `## Agent Team` exists, it extracts each subsection
3. `### all` rules are appended to **every** teammate's prompt
4. `### <role>` rules are appended to the **corresponding** teammate's prompt
5. If the section doesn't exist, no project-specific rules apply — the flow works without it

### Supported roles

| Section | Injected into |
|---------|---------------|
| `### all` | coder, reviewer, senior-reviewer, final-reviewer |
| `### coder` | coder only |
| `### reviewer` | reviewer only |
| `### senior-reviewer` | senior-reviewer only |
| `### final-reviewer` | final-reviewer only |

## Key features

| Feature | Description |
|---------|-------------|
| Autonomous pipeline | Issue → branch → code → 3 reviews → merge, no manual steps except final confirmation |
| 5-agent architecture | Team lead + coder + reviewer + senior-reviewer + final-reviewer with strict permissions |
| Kill + respawn reviews | Each review iteration uses a fresh agent to avoid confirmation bias |
| 3-category findings | Blockers / Recommended / Minor — no other categories allowed |
| Coder progress reports | Per-step updates via SendMessage so the team lead tracks implementation progress |
| Pre-flight checks | Verifies gh auth, git remote, and base branch existence before starting |
| Error recovery | Crashed agents re-spawned automatically (max 2 retries) |
| Git worktrees | Each team gets its own directory — parallel teams never interfere with each other |
| Parametrized branches | `$BASE_BRANCH`, `$BRANCH`, and `$WORKTREE` variables used in all git commands |
| Branch naming | `feature/<issue-number>-<name>-<short-hash>` — readable + unique to avoid collisions |
| Review summary | Iteration counts, findings per stage, total resolved — printed before merge |
| Project-specific rules | Per-role instructions via `CLAUDE.md` `## Agent Team` section |
| Sequential multi-issue | Multiple issues processed one by one, pulling latest base between each |
| Auto-merge flag | Skip confirmation when all reviews pass with zero findings |
| Base branch flag | Target any branch instead of the default `staging` |

## Related: Skills

All reviewers use `/pr-review`, a custom Claude Code skill defined in a separate repo:

- **Repo**: [fernandezdiegoh/df-claude-skills](https://github.com/fernandezdiegoh/df-claude-skills)
- **Skill**: `pr-review` (v2.1.0+, invoked with `--team` flag)

The `--team` flag ensures reviewers deliver findings via SendMessage (not GitHub comments) and never execute `gh` write commands. See the [pr-review README](https://github.com/fernandezdiegoh/df-claude-skills/tree/main/skills/pr-review) for details.

```bash
git clone git@github.com:fernandezdiegoh/df-claude-skills.git
```

## Version history

| Version | Changes |
|---------|---------|
| 2.5.0 | Mandatory test re-run after reviewer fixes, rebase before merge, cleanup on failure, documented `ls` workaround |
| 2.4.0 | Git worktrees for parallel-safe execution, all git commands via `git -C "$WORKTREE"` |
| 2.3.0 | Renamed invocation to `agent-team`, unique branch names with random hash |
| 2.2.0 | `--base` parameter, pre-flight checks, error recovery, coder progress checkpoints, parametrized `$BRANCH`/`$BASE_BRANCH`, branch naming with issue number, generic Co-Authored-By |
| 2.1.0 | `--team` flag on all `/pr-review` invocations (requires pr-review v2.1.0+) |
| 2.0.0 | Final-reviewer stage, project-specific rules via `CLAUDE.md`, review summary table, `--auto-merge` flag |
| 1.2.0 | English translation, 3-category findings, mandatory TeamCreate, SendMessage delivery, kill + respawn pattern |
| 1.1.0 | README, permissions table, review loops, df-claude-skills reference |
| 1.0.0 | Initial release: team lead + coder + reviewer + senior-reviewer, sequential multi-issue, permission model |
