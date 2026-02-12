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
When the user says "team-lead <issues>", read `<path-to>/agent-teams/agent-team.md` and execute the flow described there.
Flow: team lead + coder + reviewer + senior-reviewer.
Invoked with issue numbers as arguments (e.g., `team-lead #<issue-number>` or `team-lead #<issue> #<issue> --auto-merge`).
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
| **Reviewer** | Functional review (`/pr-review --team`) | NO | NO | NO (only `gh pr view`) |
| **Senior reviewer** | Consistency review (`/pr-review --team`) | NO | NO | NO (only `gh pr view`) |
| **Final reviewer** | Cold generic review (`/pr-review --team`) | NO | NO | NO (only `gh pr view`) |

### Flow

```
PER ISSUE (sequential):
  1. Team lead reads issue, creates branch from staging
  2. Team lead spawns coder with detailed instructions
  3. Coder implements, runs tests, reports back
  4. Team lead commits, pushes, creates PR to staging
  5. Team lead spawns reviewer → review loop (functional, max 5 iter)
  6. Team lead spawns senior reviewer → review loop (consistency, max 3 iter)
  7. Team lead spawns final reviewer → review loop (cold/generic, max 3 iter)
  8. User confirms → merge → pull staging
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

**Limits:**
- **Reviewer loop**: max 5 iterations, then asks user whether to continue or hand off to human review
- **Senior reviewer loop**: max 3 iterations, same escalation
- **Final reviewer loop**: max 3 iterations, same escalation

### Merge behavior

- **Default**: asks user for confirmation before `gh pr merge`
- **`--auto-merge` flag**: merges automatically after both reviews pass with zero findings

### Multiple issues

When multiple issues are provided, they are processed **sequentially**. Each issue goes through the full flow (branch → code → review → merge) before starting the next. After each merge, staging is pulled to ensure the next issue starts from the latest base.

## Project-specific rules

The team lead automatically reads the `## Agent Team` section in your project's `CLAUDE.md` and injects per-role instructions into each teammate's prompt. This keeps `agent-team.md` generic while letting each project define its own rules.

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
```

### How it works

1. Team lead reads `CLAUDE.md` during setup (Step 2)
2. If `## Agent Team` exists, it extracts each subsection
3. `### all` rules are appended to **every** teammate's prompt
4. `### <role>` rules are appended to the **corresponding** teammate's prompt
5. If the section doesn't exist, no project-specific rules apply — the flow works without it

### Supported roles

| Section | Injected into |
|---------|---------------|
| `### all` | coder, reviewer, senior-reviewer |
| `### coder` | coder only |
| `### reviewer` | reviewer only |
| `### senior-reviewer` | senior-reviewer only |

## Files

| File | Description |
|------|-------------|
| `agent-team.md` | Full workflow definition — generic, works for any project |

## Related: Skills

The reviewer and senior-reviewer agents use `/pr-review`, a custom Claude Code skill defined in a separate repo:

- **Repo**: [fernandezdiegoh/df-claude-skills](https://github.com/fernandezdiegoh/df-claude-skills)
- **Skill used by agent-team**: `pr-review` (v2.1.0+, invoked with `--team` flag)

The `--team` flag ensures reviewers deliver findings via SendMessage (not GitHub comments) and never execute `gh` write commands. See the [pr-review README](https://github.com/fernandezdiegoh/df-claude-skills/tree/main/skills/pr-review) for details.

Clone it alongside this repo for local access:

```bash
git clone git@github.com:fernandezdiegoh/df-claude-skills.git
```
