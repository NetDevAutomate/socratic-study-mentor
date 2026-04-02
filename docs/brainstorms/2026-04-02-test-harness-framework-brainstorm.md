# Test Harness Framework for Study Command Lifecycle

**Date:** 2026-04-02
**Status:** Decided — ready to implement

## What We're Building

A modular test harness that provides a clean API for testing tmux-based study sessions end-to-end. Replaces flaky `time.sleep()` patterns with reliable polling. Extensible for future features (web UI, speech integration).

## Why This Approach

The existing integration tests have the right *idea* (real tmux, mock agents, `STUDYCTL_TEST_AGENT_CMD` injection) but the wrong *execution* (fixed sleeps, stale state leakage, no pane content assertions). Rather than patching the existing tests, we build a proper harness layer that future test modules can import.

## Key Decisions

1. **Modular harness under `tests/harness/`** — not a single fixture file
2. **Two layers**: `TmuxHarness` (generic tmux control) → `StudySession` (study-specific lifecycle)
3. **Poll-based waits everywhere** — `wait_for(predicate, timeout, msg)` replaces all `time.sleep()`
4. **Mock agents stay as bash scripts** — `STUDYCTL_TEST_AGENT_CMD` pattern is proven
5. **Context manager cleanup** — sessions are killed on `__exit__` regardless of test outcome
6. **Pytest fixture** — `study_session` fixture yields a `StudySession`, cleans up after
7. **Marked `@pytest.mark.integration`** — skipped on CI, run locally

## Architecture

```
tests/harness/
├── __init__.py
├── tmux.py         # TmuxHarness — core tmux control + polling
├── study.py        # StudySession — study lifecycle API
└── agents.py       # Mock agent script builders
```

### TmuxHarness (tmux.py)
- `wait_for(predicate, timeout=15, interval=0.5, msg="")` — reliable polling
- `capture_pane(pane_id)` → str — get scrollback content
- `wait_for_pane_content(pane_id, pattern, timeout=15)` → str — poll until match
- `pane_has_children(pane_id)` → bool
- `session_exists(name)` → bool
- `kill_all_study_sessions()` — cleanup helper

### StudySession (study.py)
- `start(topic, energy=5, agent_cmd=None)` — create session, wait for ready
- `resume()` — run --resume, wait for ready
- `end_via_Q()` — press Q in sidebar, wait for session to die
- `end_via_cli()` — run studyctl study --end
- `kill_agent()` — simulate agent crash (kill child process)
- `assert_agent_running()` / `assert_sidebar_running()`
- `assert_pane_contains(pane, pattern, timeout=10)`
- `assert_session_ended()`
- Properties: `session_name`, `main_pane`, `sidebar_pane`, `state`
- Automatic cleanup via context manager / fixture teardown

### MockAgent builders (agents.py)
- `long_running()` — stays alive, loops forever
- `topic_logger(topics)` — logs specific topics via studyctl CLI, stays alive
- `fast_exit()` — logs one topic, exits immediately
- `parking_agent()` — parks questions via studyctl park

## Test Module: test_study_lifecycle.py

```python
class TestSessionStart:
    def test_creates_tmux_with_two_panes(study_session)
    def test_agent_pane_shows_prompt(study_session)
    def test_sidebar_pane_shows_timer(study_session)
    def test_state_file_written(study_session)

class TestSidebarUpdates:
    def test_topics_appear_in_sidebar(study_session)
    def test_parking_appears_in_sidebar(study_session)
    def test_counters_update(study_session)

class TestQuitFlow:
    def test_Q_kills_agent_and_tmux(study_session)
    def test_state_set_to_ended_after_Q(study_session)

class TestResumeFlow:
    def test_resume_reconnects_to_live_session(study_session)
    def test_resume_rebuilds_after_zombie(study_session)
    def test_resume_launches_agent_with_r_flag(study_session)
```

## Open Questions

None — design is clear. Build it.
