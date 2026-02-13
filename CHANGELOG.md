# Changelog

All notable changes to this project will be documented in this file.

## [Unreleased]

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
