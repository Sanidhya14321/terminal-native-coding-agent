# Terminal-Native Coding Agent

> **CLI in, pull request out.** A terminal-native coding agent that executes a plan-act-observe-recover loop, evaluated against SWE-bench Pro.

[![Bun](https://img.shields.io/badge/Bun-1.2+-000000?logo=bun)](https://bun.sh)
[![TypeScript](https://img.shields.io/badge/TypeScript-5.5+-3178C6?logo=typescript)](https://www.typescriptlang.org/)
[![Next.js](https://img.shields.io/badge/Next.js-16-000000?logo=next.js)](https://nextjs.org/)
[![License](https://img.shields.io/badge/license-MIT-blue)](LICENSE)

## 🎯 Overview

This agent takes a natural-language task description and a GitHub repository as input, and produces a pull request as output. It runs entirely in your terminal via a split-pane TUI built with **Ink 5** (React for terminals), or in headless mode for CI/CD pipelines.

### Key Features

| Feature | Description |
|---------|-------------|
| **Plan-Act-Observe-Recover Loop** | Strictly typed TodoWrite schema; model rewrites full plan state each turn |
| **Model-Agnostic** | OpenRouter unified API with fallback chain: Claude Sonnet 4.7 → GPT-5.4-Codex → Gemini 3 Pro → Opus 4.5 |
| **Sandbox Isolation** | Fresh E2B or Daytona sandbox per task; `git worktree` isolation; host filesystem unreachable |
| **MCP Tool Transport** | StreamableHTTP server (2026 revision) with swappable in-process vs HTTP transport |
| **Hard Budget Cutoffs** | 50 turns, 200K tokens, $5.00 per task; PreCompact summarization at 150K tokens |
| **Lifecycle Hooks** | SessionStart, PreToolUse, PostToolUse, Stop, PreCompact, SessionEnd, UserPromptSubmit, Notification |
| **Observability** | OpenTelemetry with `gen_ai.*` semantic conventions → Langfuse trace exporter |
| **Reviewer Sub-Agent** | Optional pre-PR diff review with JSON-structured feedback and revision loop |
| **Next.js Dashboard** | Dark terminal-themed web UI for run history, evaluation metrics, and agent triggering |

---

## 🏗 Architecture

```
user CLI  →  harness (Bun + Ink TUI)
                  │
                  ▼
     plan / act / observe loop  ←→  Claude / GPT / Gemini (via OpenRouter)
                  │
                  ▼
     tool dispatcher (MCP StreamableHTTP client)
                  │
     ┌────────────┼────────────┬────────────┐
     │            │            │            │
     ▼            ▼            ▼            ▼
  read_file   edit_file   ripgrep   tree_sitter
     │            │            │            │
     └────────────┴────────────┴────────────┘
                  │
                  ▼
     run_shell  ─┴─  git (status, diff, commit, push)
                  │
                  ▼
           E2B / Daytona sandbox (worktree isolated)
                  │
                  ▼
     hooks: PreToolUse / PostToolUse / Session / Prompt / Compact / Stop
                  │
                  ▼
     OpenTelemetry ── gen_ai.* ──→ Langfuse (spans, tokens, $)
                  │
                  ▼
     PR via GitHub App (plan + diff summary in body)
                  │
                  ▼
     Reviewer Sub-Agent (optional pre-PR review loop)
```

### Every Arrow, Traced to Code

See [`ARCHITECTURE_TRACE.md`](ARCHITECTURE_TRACE.md) for a line-by-line mapping of every arrow in the diagram to the exact source file, function, and line number.

---

## 📁 Directory Structure

```
terminal-native-coding-agent/
├── src/                          # Backend (Bun + TypeScript)
│   ├── cli.tsx                   # CLI entry (commander + Ink TUI)
│   ├── agent/
│   │   └── harness.ts            # Main agent loop: plan-act-observe-recover
│   ├── tui/
│   │   ├── App.tsx               # Split-pane layout (Plan | Tool Stream | Budget)
│   │   ├── PlanPane.tsx          # Todo list rendering
│   │   ├── ToolStream.tsx        # Live tool-call stream
│   │   └── BudgetBar.tsx         # Token & cost budget bar
│   ├── plan/
│   │   ├── schema.ts             # Zod-typed TodoWrite schema
│   │   ├── state.ts              # Immutable state mutations
│   │   └── persistence.ts        # .agent/state.json crash recovery
│   ├── mcp/
│   │   ├── client.ts             # MCPClient abstraction (InProcess + HTTP)
│   │   ├── server.ts             # StreamableHTTP server
│   │   ├── executor.ts           # Tool-call dispatcher
│   │   ├── stdio.ts              # Stdio transport benchmark
│   │   └── tools/
│   │       └── index.ts           # 6 tools: read, edit, ripgrep, tree-sitter, shell, git
│   ├── sandbox/
│   │   ├── sandbox.ts            # Factory
│   │   ├── e2b.ts                # E2B SDK driver
│   │   ├── daytona.ts            # Daytona REST API driver
│   │   ├── worktree.ts           # git worktree add / remove lifecycle
│   │   └── types.ts              # Sandbox interfaces
│   ├── hooks/
│   │   ├── types.ts              # Hook interface (8 hooks)
│   │   ├── registry.ts           # HookRegistryInstance
│   │   ├── built-in.ts           # SessionStart, PreToolUse, PostToolUse, Stop
│   │   └── stubs.ts              # SessionEnd, UserPromptSubmit, Notification, PreCompact
│   ├── model/
│   │   ├── router.ts             # OpenRouter unified API client
│   │   ├── vllm.ts               # Qwen3-Coder-30B via vLLM
│   │   └── types.ts              # Model & chat interfaces
│   ├── cost/
│   │   ├── budget.ts             # BudgetTracker (hard cutoffs)
│   │   ├── summarizer.ts         # PreCompact LLM summarization
│   │   └── types.ts              # Budget constants & rates
│   ├── reviewer/
│   │   └── subagent.ts           # Pre-PR diff review with revision loop
│   ├── vcs/
│   │   └── github.ts             # GitHub App PR creation
│   └── otel/
│       └── setup.ts              # OpenTelemetry + gen_ai.* + Langfuse
├── frontend/                     # Next.js 16 Dashboard
│   ├── src/app/
│   │   ├── page.tsx              # Dashboard: stats + trigger form + runs
│   │   ├── runs/page.tsx         # All runs table
│   │   ├── runs/[id]/page.tsx    # Run detail (todos, logs, budget)
│   │   ├── eval/page.tsx         # Evaluation results + chart
│   │   ├── api/runs/route.ts     # GET/POST runs
│   │   ├── api/runs/[id]/route.ts  # GET single run
│   │   └── api/eval/route.ts     # GET evaluation data
│   ├── src/components/
│   │   ├── StatsCard.tsx         # Metric cards
│   │   ├── RunTable.tsx          # Runs table
│   │   ├── RunTrigger.tsx        # Agent trigger form
│   │   └── EvalChart.tsx         # SVG bar chart (turns vs cost)
│   └── src/lib/data.ts           # Data loading (results.jsonl, runs.json)
├── eval/                         # Python Evaluation Harness
│   ├── harness.py                # Main runner (30-task SWE-bench subset)
│   ├── runner.py                 # Single-task Bun invoker
│   ├── metrics.py                # pass@1, turns-per-task, $-per-task
│   ├── swe_bench_loader.py       # Subset loader
│   ├── requirements.txt          # Python deps
│   └── results.jsonl             # 30-task evaluation trace
├── test/                         # Unit Tests (Bun test runner)
│   ├── cost-control.test.ts      # Budget, PreCompact, model routing
│   ├── red-team.test.ts          # Sandbox destructive-command guards
│   ├── reviewer.test.ts          # Reviewer sub-agent lifecycle
│   └── transport-benchmark.test.ts # MCP HTTP vs stdio, truncation
├── package.json                  # Bun dependencies
├── tsconfig.json                 # TypeScript config
├── AGENT_SPEC.md                 # Coordination contract (swarm-coding spec)
├── ARCHITECTURE_TRACE.md        # Line-by-line diagram → code mapping
└── README.md                     # This file
```

---

## 🚀 Quick Start

### Prerequisites

- [Bun 1.3+](https://bun.sh/docs/installation) (backend runtime)
- [Node.js 20+](https://nodejs.org/) (frontend)
- [Git](https://git-scm.com/)

### Environment Variables

```bash
export OPENROUTER_API_KEY="sk-or-..."           # Required for model routing
export E2B_API_KEY="e2b-..."                    # For E2B sandbox (or use Daytona)
export DAYTONA_API_KEY="..."                     # Alternative sandbox provider
export GITHUB_TOKEN="ghp_..."                    # For PR creation (or GitHub App)
export OTEL_EXPORTER_OTLP_ENDPOINT="http://localhost:4318/v1/traces"  # Langfuse
```

### Backend

```bash
# Install
bun install

# Build
bun run build

# Test
bun test

# Run with TUI (interactive terminal)
bun run cli run django/django "Fix admin module bug"

# Headless mode (CI/CD)
bun run cli run django/django "Fix admin module bug" --no-tui

# With reviewer sub-agent
bun run cli run django/django "Fix admin module bug" --reviewer

# Resume after crash
bun run cli run django/django "Fix admin module bug" --resume
```

### Frontend Dashboard

```bash
cd frontend
npm install
npm run dev        # http://localhost:3000
```

Pages:
- `/` — Dashboard with stats, agent trigger form, recent runs
- `/runs` — All runs table
- `/runs/[id]` — Run detail (todos, logs, budget, PR link)
- `/eval` — Evaluation results with bar chart and metrics

---

## 🧪 Evaluation

The Python harness evaluates the agent on a 30-issue subset of SWE-bench Pro.

```bash
cd eval
pip install -r requirements.txt

# Dry run (load subset without executing)
python harness.py --dry-run --subset-size 30

# Full evaluation
python harness.py --subset-size 30 --output results.jsonl

# Compare against baseline
python harness.py --subset-size 30 --output results.jsonl --compare-baseline baseline.jsonl
```

### Output

- `eval/results.jsonl` — Per-task JSON lines with: `instance_id`, `success`, `turns`, `tokens`, `cost`, `pr_url`, `error`
- `eval/results.summary.json` — Aggregated metrics: `pass@1`, `avg_turns`, `avg_cost`, `median_turns`, `median_cost`

### Mock Results (30 tasks)

| Metric | Value |
|--------|-------|
| pass@1 | 66.7% (20/30 passed) |
| Avg Turns | 18.6 |
| Median Turns | 15.0 |
| Avg Cost | $0.93 |
| Median Cost | $0.45 |
| Avg Tokens | 62,400 |

---

## 🛠 Tool Surface (6 MCP Tools)

Every tool returns output truncated to **4,000 tokens** with `truncated: true` flag.

| Tool | Args | Description |
|------|------|-------------|
| `read_file` | `path` | Read file from worktree |
| `edit_file` | `path`, `old_string`, `new_string` | Edit file with diff preview |
| `ripgrep` | `pattern`, `path` | Search code with `rg` subprocess |
| `tree_sitter_symbols` | `path` | Extract symbols (Python AST fallback) |
| `run_shell` | `command`, `timeout` | Execute shell with timeout |
| `git` | `subcommand`, `args` | `status`, `diff`, `commit`, `push` |
| `todo_write` | `todos[]` | Rewrite full plan state (id, status, title, notes) |

---

## 🔐 Lifecycle Hooks (2026 Architecture)

| Hook | Status | Purpose |
|------|--------|---------|
| `SessionStart` | ✅ Implemented | Initialize task budget, OTel span |
| `PreToolUse` | ✅ Implemented | Destructive-command guard (blocks `rm -rf`, `mkfs`, writes outside worktree) |
| `PostToolUse` | ✅ Implemented | Per-turn token accounting, duration logging |
| `Stop` | ✅ Implemented | Export OpenTelemetry trace bundle |
| `PreCompact` | ✅ Implemented | Summarize older turns at 150K token threshold |
| `SessionEnd` | ✅ Stub | Fires on Ctrl-C before exit |
| `UserPromptSubmit` | ✅ Stub | Reserved for future use |
| `Notification` | ✅ Stub | Reserved for future use |

---

## 📊 Budget Hard Cutoffs

| Resource | Limit | PreCompact Trigger |
|----------|-------|-------------------|
| Turns | 50 | — |
| Tokens | 200,000 | 150,000 |
| Dollars | $5.00 | — |

The `BudgetTracker` class enforces these cutoffs and triggers `PreCompact` at the threshold.

---

## 🔭 Observability (OpenTelemetry → Langfuse)

Every task generates an OpenTelemetry trace with `gen_ai.*` semantic conventions:

| Attribute | Value |
|-----------|-------|
| `gen_ai.system` | `"openrouter"` |
| `gen_ai.request.model` | Requested model |
| `gen_ai.response.model` | Actual model (after fallback) |
| `gen_ai.usage.input_tokens` | Prompt tokens |
| `gen_ai.usage.output_tokens` | Completion tokens |
| `gen_ai.operation.name` | `"chat.completions"` |

Set `OTEL_EXPORTER_OTLP_ENDPOINT` to your Langfuse collector.

---

## 🔄 Model Swap (Post-MVP)

### Qwen3-Coder-30B via vLLM

```typescript
import { createVLLMRouter } from "src/model/vllm.js";

const router = createVLLMRouter();  // Uses VLLM_BASE_URL, VLLM_MODEL env vars
```

### Benchmark: MCP StreamableHTTP vs stdio

```bash
bun test test/transport-benchmark.test.ts
```

Measured overhead: **~6x** for HTTP serialization vs direct in-process dispatch.

---

## 🧪 Tests

```bash
bun test
```

| Test File | Tests | Coverage |
|-----------|-------|----------|
| `cost-control.test.ts` | 6 | Budget limits, PreCompact, model routing, cost estimation |
| `red-team.test.ts` | 3 | Destructive-command guards, worktree escape, network isolation |
| `reviewer.test.ts` | 3 | Sub-agent initialization, diff review, revision loop |
| `transport-benchmark.test.ts` | 4 | HTTP vs stdio, tool truncation, server health |

**16 tests, 0 failures, 226 expect() calls**

---

## 📝 Specification

- [`AGENT_SPEC.md`](AGENT_SPEC.md) — Original coordination contract with subsystem breakdown, shared contracts, and merge order
- [`ARCHITECTURE_TRACE.md`](ARCHITECTURE_TRACE.md) — Every diagram arrow mapped to exact file, function, and line number

---

## 🏛 Stack

| Layer | Technology |
|-------|------------|
| Harness Runtime | Bun 1.3 + Ink 5 (React-in-terminal) |
| Model Routing | OpenRouter unified API |
| Tool Transport | MCP StreamableHTTP (2026 revision) |
| Sandbox | E2B (JS SDK) or Daytona devcontainers |
| Code Search | ripgrep + tree-sitter (17 languages) |
| Eval Scripts | Python 3 |
| Observability | OpenTelemetry SDK + `gen_ai.*` → Langfuse |
| VCS | GitHub App with fine-grained tokens |
| Frontend | Next.js 16 + Tailwind CSS |
| Testing | Bun test runner |

---

## 🤝 Contributing

To contribute:

1. Fork the repository
2. Create a feature branch (`git checkout -b feature/amazing-feature`)
3. Commit your changes (`git commit -m 'feat: add amazing feature'`)
4. Push to the branch (`git push origin feature/amazing-feature`)
5. Open a Pull Request

---

## 📄 License

MIT © Sanidhya Vats

---

<p align="center">
  <sub>Built with ❤️ using Bun, Ink, and the swarm-coding methodology.</sub>
</p>
