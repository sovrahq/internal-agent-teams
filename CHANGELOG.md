# Changelog

All notable changes to this project will be documented in this file.

## [Unreleased]

## 2026-02-13

- Fixed concurrent team branch safety: exact branch name check replaces `feature/*` glob (prevents Team A committing on Team B's branch)
- Added mandatory branch verification before every git operation (add, commit, push), including in review loop
- Added CONCURRENT TEAMS WARNING in Step 1 with branch name storage

## 2026-02-12

- Added `--team` flag to all `/pr-review` invocations (requires pr-review v2.1.0+)
- Updated README: permissions table, review loops, and skills section reflect `--team` flag

## 2026-02-11

- Added final-reviewer stage: cold generic review with no scope restrictions (max 3 iterations)
- Added project-specific rules via `CLAUDE.md` (`## Agent Team` section with per-role subsections)
- Added review summary table before merge (iterations, findings per stage, total resolved)
- Added `--auto-merge` flag: skip user confirmation when all reviews pass with zero findings
- Removed manual test metrics tracking from agent flow (delegated to `/codebase-audit`)

## 2026-02-10

- Translated `agent-team.md` to English
- Enforced strict 3-category classification for reviewer findings (Blockers / Recommended / Minor only)
- Made TeamCreate mandatory in Step 1 (teammates invisible without it)
- Added SendMessage instructions for all reviewers (findings must go via SendMessage, not text output)
- Added kill + respawn pattern for reviewers (avoids confirmation bias between iterations)

## 2026-02-09

- Added comprehensive README with flow diagram, permissions table, review loops, and merge behavior
- Added reference to df-claude-skills repo (pr-review skill)
- Removed hardcoded paths, standardized to `#issue` format in examples

## 2026-02-08

- Initial release: team lead + coder + reviewer + senior-reviewer workflow
- Sequential multi-issue processing
- Permission model: team lead owns git/gh, coder edits files, reviewers read-only
