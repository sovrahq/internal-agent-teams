# Changelog

All notable changes to this project will be documented in this file.

## [Unreleased]

## 2.6.1

- Fixed reviewer worktree access: all reviewer prompts now include `$WORKTREE` path so reviewers read files from the correct branch instead of attempting `git checkout` (which fails because the branch is locked by the worktree)

## 2.6.0

- By default, only Blockers and Recommended improvements must be resolved. Minor suggestions are reported but do not block approval.
- Added `--strict` flag: when passed, all findings including Minor suggestions must be resolved (previous default behavior)
- Review loop exit condition updated: 0 Blockers + 0 Recommended (or 0 total if `--strict`)

## 2.5.1

- Fixed merge cleanup order: remove worktree BEFORE `gh pr merge --delete-branch` (branch deletion fails while worktree exists)

## 2.5.0

- Coder must re-run tests after fixing reviewer findings before team lead commits
- Added rebase on `$BASE_BRANCH` before merge to catch conflicts early (with `--force-with-lease` push)
- Added cleanup-on-failure section: worktree removal + branch deletion + TeamDelete on abort
- Documented why `ls -la` is used instead of Glob/git status (Claude Code tool caching in worktrees)

## 2.4.0

- Switched to git worktrees: each team gets its own isolated directory (`$WORKTREE`), enabling safe parallel execution
- All git commands now use `git -C "$WORKTREE"` to operate inside the worktree
- Post-merge cleanup: `git worktree remove` + `git branch -d` instead of `git checkout $BASE_BRANCH`
- Removed dirty working tree pre-flight check (no longer needed with worktrees)

## 2.3.0

- Renamed invocation command from `team-lead` to `agent-team`
- Changed branch naming to `feature/<issue>-<name>-<hash>` with random hash to avoid collisions on re-runs and parallel teams
- Added dirty working tree check to pre-flight (Step 0 now fails if uncommitted changes exist)

## 2.2.0

- Added `--base <branch>` parameter to configure base branch (defaults to `staging`)
- Added Step 0 pre-flight check (gh auth, git remote, base branch verification)
- Added final-reviewer to teammate count and project-specific rules sections
- Changed branch naming to `feature/<issue-number>-<descriptive-name>` format
- Parametrized all git commands with `$BRANCH` and `$BASE_BRANCH` variables
- Added Error Recovery section for crashed/unresponsive teammates
- Changed Co-Authored-By to generic `Claude <noreply@anthropic.com>`
- Added coder progress checkpoints (reports per numbered step via SendMessage)
- Fixed "both reviews" → "all 3 reviews" in `--auto-merge` description
- Fixed hardcoded "staging or main" → `$BASE_BRANCH` in branch safety check
- Rewrote README to match df-claude-skills quality standard (examples, feature table, version history)

## 2.1.0

- Added `--team` flag to all `/pr-review` invocations (requires pr-review v2.1.0+)
- Updated README: permissions table, review loops, and skills section reflect `--team` flag

## 2.0.0

- Added final-reviewer stage: cold generic review with no scope restrictions (max 3 iterations)
- Added project-specific rules via `CLAUDE.md` (`## Agent Team` section with per-role subsections)
- Added review summary table before merge (iterations, findings per stage, total resolved)
- Added `--auto-merge` flag: skip user confirmation when all reviews pass with zero findings
- Removed manual test metrics tracking from agent flow (delegated to `/codebase-audit`)

## 1.2.0

- Translated `agent-team.md` to English
- Enforced strict 3-category classification for reviewer findings (Blockers / Recommended / Minor only)
- Made TeamCreate mandatory in Step 1 (teammates invisible without it)
- Added SendMessage instructions for all reviewers (findings must go via SendMessage, not text output)
- Added kill + respawn pattern for reviewers (avoids confirmation bias between iterations)

## 1.1.0

- Added comprehensive README with flow diagram, permissions table, review loops, and merge behavior
- Added reference to df-claude-skills repo (pr-review skill)
- Removed hardcoded paths, standardized to `#issue` format in examples

## 1.0.0

- Initial release: team lead + coder + reviewer + senior-reviewer workflow
- Sequential multi-issue processing
- Permission model: team lead owns git/gh, coder edits files, reviewers read-only
