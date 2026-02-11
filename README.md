# Agent Teams

Reusable agent team definitions for coordinating Claude Code sessions across projects.

## Setup

Clone this repo alongside your projects:

```bash
git clone git@github.com:sovrahq/agent-teams.git
```

Add this to your project's Claude Code memory (MEMORY.md or CLAUDE.md), adjusting the path to where you cloned the repo:

```
## Agent Team (swarm)
Cuando el usuario diga "team-lead <issues>", leé `<path-to>/agent-teams/agent-team.md` y ejecutá el flujo descrito ahí.
Flujo: team lead + coder + reviewer + senior-reviewer.
Se invoca con issue numbers como argumento (ej: `team-lead #<issue-number>` o `team-lead #<issue> #<issue> --auto-merge`).
```

Then invoke from any Claude Code session:

```
team-lead #<issue-number>
team-lead #<issue> #<issue> --auto-merge
```

## How it works

The main Claude Code session reads `agent-team.md` and acts as **team lead**, using `TeamCreate`, `Task`, and `SendMessage` to spawn and coordinate teammates.

### Agents and permissions

| Agent | Role | Edit files | Run tests | Git/gh |
|-------|------|-----------|-----------|--------|
| **Team lead** | Coordination, git, PRs | NO | NO | YES (all) |
| **Coder** | Implementation | YES | YES | NO (only `gh issue view`) |
| **Reviewer** | Functional review (`/pr-review`) | NO | NO | NO (only `gh pr view`) |
| **Senior reviewer** | Consistency review (`/pr-review`) | NO | NO | NO (only `gh pr view`) |

### Flow

```
PER ISSUE (sequential):
  1. Team lead reads issue, creates branch from staging
  2. Team lead spawns coder with detailed instructions
  3. Coder implements, runs tests, reports back
  4. Team lead commits, pushes, creates PR to staging
  5. Team lead spawns reviewer → review loop
  6. Team lead spawns senior reviewer → review loop
  7. User confirms → merge → pull staging
  8. Next issue (if any)
```

### Review loops

The review process uses a **kill + respawn** pattern to avoid confirmation bias:

1. Reviewer examines the PR using `/pr-review` and reports findings
2. Team lead **kills the reviewer** (shutdown) and forwards findings to the coder
3. Coder fixes **all** findings (blockers, improvements, AND minor suggestions)
4. Team lead commits, pushes, and **spawns a fresh reviewer** with:
   - The previous findings as context (to verify they were resolved)
   - Instructions to do a full review, not just verify fixes
5. Loop continues until a reviewer approves with **zero findings**

**Limits:**
- **Reviewer loop**: max 5 iterations, then asks user whether to continue or hand off to human review
- **Senior reviewer loop**: max 3 iterations, same escalation

### Merge behavior

- **Default**: asks user for confirmation before `gh pr merge`
- **`--auto-merge` flag**: merges automatically after both reviews pass with zero findings

### Multiple issues

When multiple issues are provided, they are processed **sequentially**. Each issue goes through the full flow (branch → code → review → merge) before starting the next. After each merge, staging is pulled to ensure the next issue starts from the latest base.

## Files

| File | Description |
|------|-------------|
| `agent-team.md` | Full workflow definition with all rules, review loops, permissions, and doc update requirements |

## Related: Skills

The reviewer and senior-reviewer agents use `/pr-review`, a custom Claude Code skill defined in a separate repo:

- **Repo**: [fernandezdiegoh/df-claude-skills](https://github.com/fernandezdiegoh/df-claude-skills)
- **Skill used by agent-team**: `pr-review`

Clone it alongside this repo for local access:

```bash
git clone git@github.com:fernandezdiegoh/df-claude-skills.git
```
