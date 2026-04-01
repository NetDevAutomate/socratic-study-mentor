# Test Consolidation & TDD Foundation

**Date**: 2026-04-01
**Status**: Decided
**Next**: `/ce:plan` for cleanup work; TDD learning via fresh project

---

## What We're Building

A consolidated, well-documented test suite for socratic-study-mentor, plus a learning path for TDD/pytest skills.

**NOT a framework from scratch** — extracting and documenting patterns that already exist in 650 tests across two packages.

## Why This Approach

### The Problem

- **650 tests exist** (293 studyctl + 357 agent-session-tools) but the user can't maintain them — they were AI-generated
- **Near-zero pytest experience** — fixtures, parametrize, markers, conftest patterns are unfamiliar territory
- **Inconsistent patterns**: 3 different mocking styles, 7 copy-pasted `migrated_db` fixtures, no pytest config for studyctl
- **AuDHD users waiting** for the app — can't pause features for a learning sabbatical
- **conftest prohibition** in studyctl/tests (pluggy conflict with agent-session-tools) forces inline fixtures

### Rejected Approaches

1. **"Build comprehensive test framework first"** — Rejected. This is Big Design Up Front for tests. YAGNI applies to test infrastructure. You can't design fixtures for features that don't exist yet.

2. **"TDD bootcamp on this codebase"** — Rejected for now. Retrofitting TDD onto 650 AI-generated tests is the hardest way to learn. Better to learn TDD on a fresh project where every decision is yours.

3. **"Document conventions, then apply"** — Rejected as primary approach. Risks the same "framework first" trap. Conventions emerge from practice.

### Chosen Approach: Three-Part Split

1. **Cleanup (this session)**: Mechanical consolidation — dedup fixtures, merge worktree tests, add pytest config, write TESTING.md. Claude does the work with heavy teaching commentary.

2. **Feature work continues**: New features get properly tested by Claude, with inline teaching moments explaining every test pattern used. User learns by active observation.

3. **TDD from scratch (future)**: User starts a new project using TDD from day one. Fresh codebase, full ownership, concepts grounded by patterns observed here.

## Key Decisions

| Decision | Choice | Rationale |
|----------|--------|-----------|
| Learn TDD on this codebase? | No — fresh project later | Retrofitting is the hardest way to learn |
| Build test framework first? | No — extract from existing | YAGNI; emergent architecture over BDUF |
| Include worktree tests? | Yes | 7 new files with useful patterns (tmux integration, session state, parking lot) |
| Who does the cleanup? | Claude, with teaching commentary | User learns by watching, ships features in parallel |
| conftest for studyctl? | Still no — use test helpers module instead | Pluggy conflict is real; a `tests/_helpers.py` can share fixtures without conftest |
| Mock convention? | `monkeypatch` for attrs/env, `patch` for functions | Document when to use which; standardise across both packages |

## Current State Audit

### What works
- 650 tests, 0 collection errors (after `uv sync --all-packages`)
- agent-session-tools has clean conftest with 5 shared fixtures
- `--import-mode=importlib` at root prevents pluggy conflicts
- `pytest.importorskip()` pattern for optional deps is clean
- Exporter tests use real fake filesystems (good integration pattern)

### What needs fixing
- 7 identical `migrated_db` fixtures across exporter tests → move to conftest
- studyctl has no `[tool.pytest.ini_options]` in its pyproject.toml
- `pytest.mark.live` declared but never used → either use it or remove it
- 3 mocking styles with no convention for when to use which
- 7 worktree test files need merging with conventions applied
- No TESTING.md documenting patterns for contributors

### What's intentional (don't change)
- No conftest.py in studyctl/tests (pluggy conflict)
- Inline fixtures in studyctl tests (consequence of above)
- `pytest.importorskip()` at module level for optional deps
- `--import-mode=importlib` (load-bearing for workspace)

## Cleanup Scope

### Phase 1: Mechanical (Claude does this)
- [ ] Deduplicate `migrated_db` fixture into agent-session-tools conftest
- [ ] Add `[tool.pytest.ini_options]` to studyctl's pyproject.toml
- [ ] Resolve `pytest.mark.live` — use it or remove it
- [ ] Merge 7 worktree test files into main tree
- [ ] Create `packages/studyctl/tests/_helpers.py` for shared fixture functions (not conftest)
- [ ] Write `docs/TESTING.md` with conventions

### Phase 2: Ongoing (with features)
- [ ] Every new feature gets TDD-style tests with teaching commentary
- [ ] Teaching moments saved to Obsidian for reference

### Phase 3: User's learning path (separate)
- [ ] Start fresh project using TDD from scratch
- [ ] Study-Mentor session on pytest/TDD fundamentals

## Open Questions

*None — all resolved during brainstorming dialogue.*

## Success Criteria

1. All 650+ tests pass from workspace root with `uv run pytest`
2. Zero duplicated fixtures
3. TESTING.md exists with clear conventions for: fixture patterns, mock conventions, test tiers, how to run tests
4. User can read any test file and understand what it does (teaching moments bridge the gap)
5. New features get properly tested with patterns explained inline
