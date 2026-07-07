> [!IMPORTANT]
> **SpexFlow is no longer maintained.** Its ideas continue — and go further — in **[SpexCode](https://github.com/shuxueshuxue/spexcode)**: where SpexFlow assembled context and a spec one feature at a time, SpexCode keeps a versioned spec tree inside your git repo and dispatches AI agents to build against it. Head there for active development.

![SpexFlow](docs/images/banner.png)

[English](README.md) | [简体中文](README.zh.md)

SpexFlow is a visual context/spec workflow tool built on React Flow. It helps you turn a concrete feature request into:

1) curated repo context (via code search or manual selection), then  
2) a high-quality implementation plan/spec from an LLM, then  
3) a clean prompt you can paste into a "fresh-context" code agent (Codex / Claude Code / etc.).

It's optimized for **"finish one well-defined feature in one shot"** rather than "keep everything in your head".

## Screenshots

**A minimal workflow** (instruction → code-search → context → LLM):

![SpexFlow minimal workflow](docs/images/specflow-minimal-workflow.png)

**A larger canvas** where you keep reusable context blocks and rerun only the parts that changed:

<img width="2770" height="1696" alt="image" src="https://github.com/user-attachments/assets/c1fd8f25-da1f-4d5b-85ed-73cabfceee65" />

**Two hops searching and planning**

<img width="2758" height="1690" alt="image" src="https://github.com/user-attachments/assets/eebb0f6f-9c56-4258-ba2c-f3ab18058e0d" />

**Spec dashboard** for managing spec runs and capturing outputs:

![Spec dashboard](docs/images/spec-dashboard.png)

## What You Build With It

SpexFlow loads a local code repo and lets you run a small node-based workflow:

- **Instruction** → produce/compose plain text input
- **Code Search Conductor** → generate multiple complementary search queries (one per downstream Code Search node)
- **Code Search** (Relace fast agentic search) → returns `{ explanation, files }`
- **Manual Import** → select files/folders and produce the same `{ explanation, files }` shape (no external search)
- **Context Converter** → turns file ranges into line-numbered text context
- **LLM** → takes context + prompt and generates an output (spec/plan/etc.)

## Prerequisites

- Node.js 18+
- pnpm 9+

## Quick Start

### 1) Install

```bash
pnpm install
```

### 2) Run

```bash
pnpm dev
```

- Web UI: open Vite dev server (printed in terminal, typically `http://localhost:5173`)
- Server health: `curl http://localhost:3001/api/health`

### 3) Configure keys (recommended)

Open **Settings** (top-right) and set:

- **Code Search**: Relace API key ([get one here](https://docs.relace.ai/docs/introduction))
- **LLM providers/models**: add at least one model under a provider with an OpenAI-compatible endpoint

### 4) Run a minimal workflow

Use the default canvas, or build:

```
instruction → code-search → context-converter → llm
```

Then copy the LLM output and paste it into your coding agent.

## Core Concepts

### Canvas = cached pipeline

- A canvas is a directed graph (nodes + edges).
- Each node has **inputs** (edges into it) and **output** (stored on the node).
- Node outputs are persisted locally in `data.json` so you can reuse them and rerun only the stale pieces.

### Run vs Chain

- **Run**: executes one node. If the node has any incoming edges, all predecessors must be `success`.
- **Chain**: executes the whole downstream subgraph from a node, respecting dependencies, and shows progress in **Chain Manager**.

### Locked / Muted

- **Locked**: node cannot be dragged and won't be reset by Chain; useful for "stable cached context".
- **Muted**: node returns empty output immediately (no API calls); useful for temporarily disabling branches.

### Spec Dashboard

- Specs live outside the canvas and reference existing nodes by ID (input + outputs).
- Running a spec injects its content into the input node, then chain-runs from there.
- Outputs are collected from the selected nodes and stored in per-spec run history.
- Star a spec as a **template** to create new specs with the same input/output mapping.

## Node Types

### `instruction`

- Purpose: write your feature request / constraints / acceptance criteria.
- Input: optional predecessor text nodes.
- Output: a single composed string (predecessor text + your typed text).

### `code-search-conductor`

- Purpose: generate multiple complementary search queries for several downstream Code Search nodes.
- Input: optional predecessor text nodes + its own query field.
- Output: JSON mapping `successor_node_id -> query`.
- Requirement: must have at least one **direct successor** `code-search` node (it assigns queries by node id).

### `code-search`

- Purpose: use Relace fast agentic search to find relevant code.
- Config:
  - `repoPath`: absolute path or relative to this project directory
  - `query`: natural language query
  - `debugMessages`: dumps full raw tool conversation to `logs/relace-search-runs/<runId>.json`
- Output shape (shared with Manual Import):
  - `explanation: string`
  - `files: Record<relPath, Array<[startLine,endLine]>>`

### `manual-import` (Manual Import)

- Purpose: hand-pick local files/folders as context (no external search).
- Config:
  - `repoPath`
  - `items`: selected files and folders (stored as relative paths; **contents are never persisted**)
- Folder behavior:
  - **Non-recursive**: includes only direct child files (one level).
  - Filters by a hardcoded "trusted extensions" allowlist (includes `.md`) in `server/repoBrowser.ts`.
- Run behavior:
  - Validates every selected path at run time; if a file/folder no longer exists, the node fails loudly.
- Output: identical shape to `code-search` so downstream Context Converter can reuse the same path/range format.

### `context-converter`

- Purpose: turn `{ explanation, files }` into a single line-numbered context string.
- Input: one or more `code-search` / `manual-import` / `context-converter` predecessors.
- Config: `fullFile` (full files) vs ranges.
- Behavior: merges and deduplicates overlapping/adjacent line ranges across all predecessors (per repo) before building context.
- UI: shows the merged file ranges in the sidebar ("Merged File Ranges").
- Output: a single string, joining multiple predecessor contexts with `---`.

### `llm`

- Purpose: run a chat-completions style LLM call over the composed prompt.
- Input: optional predecessor text nodes.
- Config:
  - `model` (selected from Settings)
  - `systemPrompt` (optional)
  - `query`
- Output: a single string.

## Node Types & Connection Rules

The app enforces a connection matrix (invalid edges are rejected). The current rules:

| Source (output) ↓ \\ Target (input) → | instruction | code-search-conductor | manual-import | code-search | context-converter | llm |
|--------------------------------------|:-----------:|:---------------------:|:------------:|:-----------:|:-----------------:|:---:|
| **instruction**                      | ✅          | ✅                    | ❌           | ✅          | ❌                | ✅  |
| **code-search-conductor**            | ❌          | ❌                    | ❌           | ✅          | ❌                | ❌  |
| **manual-import**                    | ❌          | ❌                    | ❌           | ❌          | ✅                | ❌  |
| **code-search**                      | ❌          | ❌                    | ❌           | ❌          | ✅                | ❌  |
| **context-converter**                | ✅          | ✅                    | ❌           | ✅          | ✅                | ✅  |
| **llm**                              | ✅          | ✅                    | ❌           | ✅          | ❌                | ✅  |

## UI Guide

### Toolbar

- **Hand Mode**: pan the canvas (`H`, or hold `Space` temporarily)
- **Select Mode**: select nodes, drag to box-select (`V`)
- **Add nodes**: Code Search / Manual Import / Search Conductor / Context / Instruction / LLM
- **Reset Canvas**: clears outputs of all unlocked nodes (must not have running nodes)

### Canvas

- Double-click a node to open its primary editor (query/text). Manual Import opens the file picker.

### Sidebar (single selection)

- **Settings** for the selected node (fields vary per node type)
- Actions:
  - **Run**: run this node
  - **Chain**: run everything downstream
  - **Reset**: clear this node's output (unless locked)
- Output:
  - Preview + "View All"
  - Copy output to clipboard

### Multi-select

- Drag-select multiple nodes → a small panel appears with **Copy** and **Delete**.
- Hotkeys:
  - `Cmd/Ctrl+C`: copy selected nodes
  - `Cmd/Ctrl+V`: paste

### Tabs

- You can keep multiple canvases as tabs.
- All tabs persist in `data.json`.

## Settings

Open **Settings** (top-right):

- **Language**: English / 中文
- **LLM Providers**:
  - Add providers with `endpoint` + `apiKey` (must be OpenAI-compatible chat-completions)
  - Add models (model id + display name)
- **Code Search**:
  - currently supports Relace

## Persistence & Files

- `data.json`: all canvases + outputs + settings (gitignored)
  - delete it to reset the app state
- `logs/relace-search.jsonl`: appended run logs (gitignored)
- `logs/relace-search-runs/<runId>.json`: optional full message dumps when `debugMessages` is enabled

## Dev / Architecture

- Frontend: Vite + React (`src/`)
- Graph UI: React Flow (`@xyflow/react`)
- Backend: Express + TypeScript (`server/`), runs via `tsx watch`
- Proxy: Vite proxies `/api` to `http://localhost:3001` (`vite.config.ts`)

## Roadmap

- [x] Code file auto merge/deduplication + visualization for context converter node output
- [ ] Export canvas to local file
- [ ] Custom LLM parameters (e.g. reasoning, temperature)
- [ ] Support local LLMs
- [ ] Token statistics
- [ ] Backup running history
- [ ] Explicit spec document management interface

## Troubleshooting

### pnpm / corepack errors

If you hit `Cannot find matching keyid` from corepack, install pnpm directly (see `AGENTS.md`).
