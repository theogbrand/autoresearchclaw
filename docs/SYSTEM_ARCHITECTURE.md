# AutoResearchClaw — System Architecture Deep Dive

## TL;DR: Not Claude Code subagents/multi-agent teams

**No, this is NOT related to Claude Code subagents or multi-agent teams.** This is a standalone Python pipeline that uses LLM API calls (OpenAI-compatible `/chat/completions`) to orchestrate a 23-stage autonomous research paper generation system. It has its own "multi-agent" abstraction built in Python — not Claude Code's Agent SDK. Claude Code is merely one possible **backend** (via the ACP adapter), not the orchestration layer.

---

## 1. Entry Points

| Entry | File | What it does |
|-------|------|-------------|
| **CLI** | `researchclaw/cli.py` → `cmd_run()` | Parses args, loads config, calls `execute_pipeline()` |
| **Python API** | `from researchclaw.pipeline.runner import execute_pipeline` | Direct programmatic access |
| **Claude Code Skill** | `.claude/skills/researchclaw/SKILL.md` | Natural language trigger → calls CLI |

---

## 2. The Pipeline Engine (Core Orchestrator)

**`researchclaw/pipeline/runner.py`** — `execute_pipeline()`

This is a **sequential state machine** that iterates through all 23 stages:

```
for stage in STAGE_SEQUENCE:
    result = execute_stage(stage, ...)
    # Handle PIVOT/REFINE decisions
    # Write checkpoints, heartbeats, knowledge base entries
```

Key mechanisms:
- **Checkpointing**: After each successful stage, writes `checkpoint.json` — supports `--resume`
- **PIVOT/REFINE loop**: Stage 15 (`RESEARCH_DECISION`) can return "pivot" (→ rollback to Stage 8) or "refine" (→ rollback to Stage 13), with artifact versioning (`stage-08_v1/`, `stage-08_v2/`)
- **Max pivots = 2** to prevent infinite loops, with quality checks before forced PROCEED
- **Iterative quality improvement**: `execute_iterative_pipeline()` re-runs stages 16-22 if quality score < threshold

---

## 3. Stage Dispatch — The Executor

**`researchclaw/pipeline/executor.py`** (~8400 lines — the largest file)

`execute_stage()` is the dispatcher. For each of the 23 stages, it:
1. Validates input artifacts exist (via `contracts.py`)
2. Creates `stage_dir` (e.g., `stage-07/`)
3. Calls the appropriate `_execute_*()` function
4. Validates output artifacts
5. Applies gate logic if needed (stages 5, 9, 20)

Each `_execute_*()` function follows the same pattern:
- Read prior artifacts from earlier stage directories
- Build a prompt using `PromptManager`
- Call `llm.chat()` — the single LLM interface
- Parse response, write artifacts to stage directory
- Return `StageResult`

---

## 4. The 23 Stages in 8 Phases

**`researchclaw/pipeline/stages.py`** — `Stage(IntEnum)`

| Phase | Stages | Key Logic |
|-------|--------|-----------|
| **A: Scoping** | 1-2 | Decomposes topic into problem tree; detects hardware (GPU/MPS/CPU) |
| **B: Literature** | 3-6 | Real API calls to OpenAlex, Semantic Scholar, arXiv; screens papers; extracts knowledge cards |
| **C: Synthesis** | 7-8 | Clusters findings, identifies gaps; **multi-agent debate** for hypothesis generation |
| **D: Design** | 9-11 | Experiment design (gate stage 9), code generation via **CodeAgent**, resource planning |
| **E: Execution** | 12-13 | Runs experiments in sandbox (local/Docker/SSH/Colab); self-healing code repair loop |
| **F: Decision** | 14-15 | Multi-agent result analysis; PROCEED/PIVOT/REFINE decision |
| **G: Writing** | 16-19 | Outline → Draft → Peer Review (multi-agent) → Revision |
| **H: Finalization** | 20-23 | Quality gate, knowledge archive, LaTeX export, citation verification |

**Gate stages** (5, 9, 20): Pause for human approval unless `--auto-approve`

---

## 5. The Three Multi-Agent Subsystems

These are Python classes using `BaseAgent`/`AgentOrchestrator` from `researchclaw/agents/base.py` — **not** Claude Code subagents.

### a) CodeAgent (`pipeline/code_agent.py`)
5-phase code generation:
1. **Blueprint Planning** — per-file pseudocode with dependency ordering
2. **Sequential File Generation** — generates files one-by-one with CodeMem
3. **Execution-in-the-Loop** — runs in sandbox, feeds errors back for repair
4. **Solution Tree Search** — explores multiple implementations, picks best
5. **Multi-Agent Review** — coder-reviewer dialogue

### b) BenchmarkAgent (`agents/benchmark_agent/orchestrator.py`)
4-agent pipeline: **Surveyor → Selector → Acquirer → Validator** (with retry loop)
- Surveys available benchmarks for the research domain
- Selects appropriate ones given hardware/time constraints
- Generates data loader + baseline code
- Validates code quality

### c) FigureAgent (`agents/figure_agent/orchestrator.py`)
Decision-based routing:
```
Decision Agent → analyzes paper → classifies figures needed
  ├── Code figures → Planner → CodeGen → Renderer → Critic (retry loop)
  └── Image figures → Nano Banana (Gemini image generation API)
→ Integrator (combines into manifest)
```
7 sub-agents: Decision, Planner, CodeGen, Renderer, Critic, NanoBanana, Integrator

---

## 6. LLM Abstraction Layer

**`researchclaw/llm/client.py`** — `LLMClient`

All LLM calls go through one interface:
- **Default**: OpenAI-compatible HTTP API (`/chat/completions`) using stdlib `urllib`
- **Anthropic adapter**: Translates to Anthropic Messages API when `provider: "anthropic"`
- **ACP adapter** (`llm/acp_client.py`): Shells out to `acpx` CLI to communicate with any AI coding agent (Claude Code, Codex, Gemini CLI, etc.) via persistent named sessions
- **MetaClaw bridge**: Routes through MetaClaw proxy for cross-run learning, with fallback to direct API

---

## 7. Adapter System (OpenClaw Bridge)

**`researchclaw/adapters.py`** — `AdapterBundle`

6 typed protocol interfaces that external platforms can implement:
- `CronAdapter` — scheduled runs
- `MessageAdapter` — notifications (Discord/Slack)
- `MemoryAdapter` — cross-session persistence
- `SessionsAdapter` — parallel sub-sessions
- `WebFetchAdapter` — live web search
- `BrowserAdapter` — browser-based paper collection

Default: `Recording*Adapter` stubs that just log calls.

---

## 8. Supporting Systems

| System | Files | Purpose |
|--------|-------|---------|
| **Literature** | `literature/` | arXiv, OpenAlex, Semantic Scholar clients; novelty checking; citation verification |
| **Experiment Sandbox** | `experiment/` | Local sandbox, Docker sandbox, SSH remote, Colab Drive — all implement same runner interface |
| **Knowledge Base** | `knowledge/` | Markdown/Obsidian KB written per-stage |
| **Evolution/Self-Learning** | `evolution.py` | Extracts lessons from each run (failures, warnings, metric anomalies) with 30-day decay |
| **MetaClaw Bridge** | `metaclaw_bridge/` | Converts lessons → reusable skills; PRM quality gates; session management |
| **Templates** | `templates/` | NeurIPS/ICML/ICLR LaTeX templates; Markdown→LaTeX converter; LaTeX compiler |
| **Prompts** | `prompts.py` + `prompts.default.yaml` | Customizable prompt templates for all 23 stages |
| **Quality** | `quality.py` | Template ratio detection, anti-fabrication guards |

---

## 9. Fastest Path to Edit for Your Use Case

To adapt this for a custom use case, these are the files that matter most, in priority order:

1. **`config.researchclaw.example.yaml`** — Change topic, LLM provider, experiment mode, templates
2. **`prompts.default.yaml`** — Customize the LLM prompts for any/all 23 stages without touching code
3. **`researchclaw/pipeline/executor.py`** — The `_execute_*()` functions contain all per-stage logic. Edit the specific stage you want to change.
4. **`researchclaw/pipeline/stages.py`** — To add/remove/reorder stages
5. **`researchclaw/pipeline/contracts.py`** — To change what artifacts each stage expects/produces
6. **`researchclaw/llm/client.py`** — To change the LLM backend
7. **`researchclaw/agents/`** — To modify the multi-agent subsystems (CodeAgent, BenchmarkAgent, FigureAgent)

---

## 10. Data Flow Diagram

```
Topic Input
    ↓
┌─────────────────────────────────────┐
│  Phase A: Research Scoping          │
│  1. TOPIC_INIT → goal.md            │
│  2. PROBLEM_DECOMPOSE → problem_tree│
└─────────────────────────────────────┘
    ↓
┌─────────────────────────────────────┐
│  Phase B: Literature Discovery      │
│  3. SEARCH_STRATEGY                 │
│  4. LITERATURE_COLLECT (OpenAlex+S2)│
│  5. LITERATURE_SCREEN [GATE]        │
│  6. KNOWLEDGE_EXTRACT → cards/      │
└─────────────────────────────────────┘
    ↓
┌─────────────────────────────────────┐
│  Phase C: Knowledge Synthesis       │
│  7. SYNTHESIS → gaps identified     │
│  8. HYPOTHESIS_GEN (multi-agent)    │
└─────────────────────────────────────┘
    ↓
┌─────────────────────────────────────┐
│  Phase D: Experiment Design         │
│  9. EXPERIMENT_DESIGN [GATE]        │ ←┐
│     (BenchmarkAgent selects data)   │  │
│  10. CODE_GENERATION (CodeAgent)    │  │
│  11. RESOURCE_PLANNING              │  │
└─────────────────────────────────────┘  │
    ↓                                    │
┌─────────────────────────────────────┐  │
│  Phase E: Experiment Execution      │  │
│  12. EXPERIMENT_RUN (sandbox)       │  │
│  13. ITERATIVE_REFINE (self-heal)   │  │
└─────────────────────────────────────┘  │
    ↓                                    │
┌─────────────────────────────────────┐  │
│  Phase F: Analysis & Decision       │  │
│  14. RESULT_ANALYSIS (multi-agent)  │  │
│  15. RESEARCH_DECISION              │──┘
│      ├─ PROCEED → continue
│      ├─ REFINE → loop to stage 13
│      └─ PIVOT → loop to stage 8
└─────────────────────────────────────┘
    ↓
┌─────────────────────────────────────┐
│  Phase G: Paper Writing             │
│  16. PAPER_OUTLINE                  │
│  17. PAPER_DRAFT (5-6.5k words)     │
│  18. PEER_REVIEW (multi-agent)      │
│  19. PAPER_REVISION                 │
└─────────────────────────────────────┘
    ↓
┌─────────────────────────────────────┐
│  Phase H: Finalization              │
│  20. QUALITY_GATE [GATE]            │
│  21. KNOWLEDGE_ARCHIVE              │
│  22. EXPORT_PUBLISH → LaTeX + figs  │
│  23. CITATION_VERIFY (4-layer)      │
└─────────────────────────────────────┘
    ↓
deliverables/ (paper.tex, refs.bib, code/, charts/)
```
