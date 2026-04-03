# Phase 2: Multi-Agent Support — Brainstorm

**Date:** 2026-04-03
**Status:** Agreed
**Phase:** 2 (Multi-Agent)
**Roadmap item:** Phase 3 deferred — agent switching mid-session

---

## What We're Building

Wire Gemini CLI, Kiro CLI, and OpenCode as first-class study session agents alongside Claude. Any installed agent can drive a `studyctl study` session with the same Socratic mentor experience — same persona, same MCP tools, same sidebar, same progress tracking.

### Scope

- **In scope:** Launch all 4 agents with persona + MCP tools, configurable auto-detection priority, doctor validation, mock + integration test harness
- **Out of scope:** Agent switching mid-session (roadmap item), ttyd/LAN support (Phase 3)

---

## Why This Approach

### Architecture: Shared Persona + Agent Adapter

```
build_canonical_persona(mode, topic, energy, backlog)
        │
        ▼
  ┌─────────────┐
  │  Canonical   │  ← Markdown content: session context + shared persona
  │  Content     │     (topic, energy, backlog, Socratic engine, CLI commands)
  └──────┬──────┘
         │
    ┌────┴────┬──────────┬──────────┐
    ▼         ▼          ▼          ▼
 Claude    Gemini      Kiro     OpenCode
 Adapter   Adapter    Adapter   Adapter
    │         │          │          │
    ▼         ▼          ▼          ▼
 --append-  --system-  agent.yml  --system-
 system-    instruction  JSON     prompt
 prompt-    flag        format    flag
 file
```

**Why adapter pattern over universal temp file:**
- Each agent has different persona injection mechanisms (CLI flags, JSON config, YAML)
- Slash commands (/) vs @commands differ — Claude uses `/` commands and runs bash directly; Kiro uses `@execute_bash`
- MCP server config is bundled differently (Kiro in agent JSON, Claude in separate mcp.json, etc.)
- Canonical content stays DRY; adapters handle serialization + agent-specific quirks

### CLI Command Execution: Wrapper Script (Already Built)

`setup_session_dir()` already creates a `studyctl` wrapper in the session directory with the correct Python/venv. `create_tmux_environment()` already sets PATH via `set_environment()`. No new work needed for command execution — just verify each agent's tmux pane inherits PATH correctly.

---

## Key Decisions

| # | Decision | Choice | Rationale |
|---|----------|--------|-----------|
| 1 | Persona injection | Shared content + per-agent adapter | DRY canonical content, agent-specific serialization handles format differences |
| 2 | Agent preference config | `~/.config/studyctl/config.toml` | Persists across sessions, fits existing settings pattern, `studyctl config set agent_priority claude,gemini` |
| 3 | Agent verification | Binary exists + smoke test | `shutil.which()` + quick command (e.g., `--help`) to catch broken installs |
| 4 | MCP wiring | Full — all agents get studyctl-mcp | Goal is identical study experience regardless of agent |
| 5 | CLI command syntax | Wrapper script approach | Already built in `setup_session_dir()`. Universal `studyctl topic ...` commands, no per-agent rewriting |
| 6 | Testing strategy | Mock unit tests + skip-if-missing integration harness | Mock-based for CI, real-binary integration tests that skip gracefully when agent not installed |
| 7 | Done criteria | All 4 launch + MCP + doctor validates | `studyctl study "topic" --agent {claude\|gemini\|kiro\|opencode}` all work, auto-detection, doctor checks |

---

## Agent Registry Design

Each agent needs these fields in `AGENT_REGISTRY`:

```python
AGENT_REGISTRY = {
    "claude": {
        "binary": "claude",
        "check": "claude --version",          # smoke test command
        "launch": "claude --append-system-prompt-file {persona_file}",
        "resume": "claude -r --append-system-prompt-file {persona_file}",
        "adapter": "claude",                   # which adapter to use
        "mcp_config": "agents/claude/mcp.json",
    },
    "gemini": {
        "binary": "gemini",
        "check": "gemini --version",
        "launch": "gemini --system-instruction {persona_file}",  # TBC
        "resume": "gemini --system-instruction {persona_file}",  # TBC
        "adapter": "gemini",
        "mcp_config": "agents/gemini/mcp.json",
    },
    "kiro": {
        "binary": "kiro",
        "check": "kiro --version",
        "launch": "kiro chat --agent study-mentor",  # TBC — Kiro loads agents natively
        "resume": "kiro chat --resume --agent study-mentor",
        "adapter": "kiro",
        "mcp_config": None,  # bundled in agent JSON
    },
    "opencode": {
        "binary": "opencode",
        "check": "opencode --version",
        "launch": "opencode --system-prompt {persona_file}",  # TBC
        "resume": "opencode --system-prompt {persona_file}",
        "adapter": "opencode",
        "mcp_config": "agents/opencode/mcp.json",
    },
}
```

**TBC items:** Exact CLI flags for Gemini, Kiro, and OpenCode need verification against current binary versions. The adapter pattern means we can adjust these without changing core logic.

---

## Adapter Interface

```python
class AgentAdapter(Protocol):
    """Transform canonical persona content for a specific agent."""

    def build_persona_file(
        self,
        canonical_content: str,
        mode: str,
        topic: str,
        energy: int,
    ) -> Path:
        """Write agent-specific persona file, return path."""
        ...

    def build_mcp_config(
        self,
        session_dir: Path,
    ) -> Path | None:
        """Write/symlink MCP config for this agent. Return path or None."""
        ...

    def get_launch_command(
        self,
        persona_file: Path,
        *,
        resume: bool = False,
    ) -> str:
        """Build the shell command to launch this agent."""
        ...

    def smoke_test(self) -> bool:
        """Quick check that the binary actually works."""
        ...
```

---

## MCP Wiring Per Agent

| Agent | MCP Mechanism | What we need to do |
|-------|--------------|-------------------|
| Claude | `mcp.json` in project dir or `~/.claude/` | Already works. Symlink/copy `agents/claude/mcp.json` to session dir |
| Gemini | `mcp.json` or GEMINI.md config | Write MCP server config in Gemini's expected format |
| Kiro | `mcpServers` in agent JSON definition | Already defined in `agents/kiro/study-mentor.json`. Ensure agent def is installed |
| OpenCode | `mcp.json` or `opencode.json` config | Write MCP config in OpenCode's expected format |

---

## Doctor Integration

Extend `doctor/agents.py` to:
1. Detect all 4 binaries (already does this)
2. Run smoke test per detected agent (new)
3. Verify agent definition installed + current (already does hash check)
4. Verify MCP config present for each detected agent (new)

---

## Open Questions

1. **Exact CLI flags** — Need to verify Gemini CLI, Kiro, and OpenCode current flags for system prompt injection. These tools evolve fast. Plan: check `--help` output when each binary is available.
2. **Kiro native agent loading** — Kiro loads agents from `~/.kiro/agents/`. The persona file approach may need to work differently: install the agent definition with session context baked in, rather than passing a CLI flag.
3. **Gemini MCP support** — Is Gemini CLI's MCP server support mature enough for studyctl-mcp? Need to verify.
4. **Config file format** — Is `config.toml` the right format, or should we use the existing settings pattern? Need to check what `settings.py` currently uses.

---

## Roadmap Items (Not Phase 2)

- **Agent switching mid-session** — Switch from Claude to Gemini within an active tmux session. Requires saving/loading conversation context. Very cool, deferred.
- **Agent-specific persona tuning** — Per-agent prompt tweaks beyond what the adapter handles (e.g., Gemini responds differently to certain prompt structures).

---

## Next Step

Run `/workflows:plan` to create the implementation plan.
