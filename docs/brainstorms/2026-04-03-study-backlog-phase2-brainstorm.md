# Brainstorm: Study Backlog Phase 2 — AI Prioritization + MCP Integration — 2026-04-03

## What We're Building

Three interconnected pieces:

1. **`studyctl topics suggest`** — CLI command that ranks pending backlog topics using frequency + agent-assessed importance
2. **4 new MCP tools** — `get_study_backlog`, `get_topic_suggestions`, `get_study_history`, `record_topic_progress` — giving the agent direct session-db access
3. **Session-db integration tests** — comprehensive tests for all session types, ensuring the full data pipeline works
4. **Architecture documentation** — updated diagrams showing session-db as a first-class component

## Scoring Model

Two signals, combined:

| Signal | Source | Weight | Rationale |
|--------|--------|--------|-----------|
| **Frequency** | COUNT of parked_topics entries per question | Data-driven | Topics that keep coming up are clearly important |
| **Importance** | Agent-assessed priority (1-5) on parked_topics | LLM-driven | Fundamental topics (OOP basics, closures) rank higher than niche ones |

**Importance scoring**: The agent sets a `priority` (1-5) when parking or struggling on a topic. The MCP `record_topic_progress` tool can also update priority. Topics without a priority score default to 3 (neutral).

**Combined score**: `score = frequency_weight * normalized_frequency + importance_weight * (priority / 5.0)`

## Key Decisions

| Decision | Choice | Rationale |
|----------|--------|-----------|
| Scoring model | Frequency + agent importance | Fundamentality matters more than recency for learning paths |
| MCP tools | All 4 (read + write) | Full agent integration, extensibility investment |
| Priority storage | `priority` INTEGER column on parked_topics | Migration v17, agent sets via MCP |
| Suggest output | Ranked list with scores + reasoning hints | Agent can re-rank with its own context |
| Session-db tests | Integration tests with real DB for all session types | Validates full data pipeline end-to-end |
| Architecture docs | Updated PlantUML with session-db as central component | Source of truth for the whole system |

## MCP Tools Spec

| Tool | Type | Description |
|------|------|-------------|
| `get_study_backlog` | Read | Query pending backlog items with filters (tech, source, status) |
| `get_topic_suggestions` | Read | Return algorithmically ranked topic suggestions |
| `get_study_history` | Read | Query study_sessions + study_progress for a topic |
| `record_topic_progress` | Write | Update study_progress confidence, record teach-back, set topic priority |

## Migration v17

```sql
ALTER TABLE parked_topics ADD COLUMN priority INTEGER;
-- NULL = not yet assessed, 1 = low, 5 = critical/foundational
```

## Scope

### In scope
- `studyctl topics suggest` CLI command (FCIS: scoring logic in backlog_logic.py)
- 4 MCP tools registered in studyctl-mcp server
- Migration v17 (priority column)
- Updated architecture doc with session-db integration diagrams
- Integration tests: migration, MCP tools, scoring, all session types

### Out of scope
- Auto-categorization of tech_area by LLM (manual or agent-set for now)
- Spaced repetition integration with suggestions (future)
