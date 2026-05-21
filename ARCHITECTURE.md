# Dr. Claw -- Architecture and Implementation Guide

> **Version:** 1.1.4  
> **Last updated:** 2026-05-21  
> **Repository:** [github.com/OpenLAIR/dr-claw](https://github.com/OpenLAIR/dr-claw)

---

## 1. Overview

### 1.1 What Dr. Claw Is

Dr. Claw is a full-stack AI research workspace that orchestrates end-to-end research projects -- from initial literature survey through ideation, experimentation, publication, and promotion. It provides a web-based UI (and desktop app) that wraps multiple AI coding agents (Claude Code, Gemini CLI, OpenAI Codex, OpenRouter) and gives them structured research "skills" (100+ prompt-engineering procedures) so they can autonomously execute multi-stage research pipelines.

The tagline "A Super AI Lab with massive AI Doctors" refers to its ability to deploy multiple AI agents (the "doctors") against a research problem, each with different capabilities and models, coordinated through a unified workspace with persistent project state.

### 1.2 High-Level Architecture

```
+-------------------+     +-------------------+     +-----------------------+
|   Frontend        |     |   Backend         |     |  Agent Integration    |
|   (React 18 +     | <-> | (Express + WS)    | <-> | (SDK / CLI wrappers)  |
|    Vite + TW CSS)  |     |                   |     |                       |
+-------------------+     +-------------------+     +-----------------------+
                                 |                          |
                          +------+------+           +-------+-------+
                          | SQLite DB   |           | Agent CLIs    |
                          | (better-    |           | - Claude Code |
                          |  sqlite3)   |           | - Gemini CLI  |
                          +-------------+           | - Codex SDK   |
                                                    | - OpenRouter  |
                                                    +---------------+
```

The system is a three-tier application:

1. **Frontend** -- React 18 SPA built with Vite, styled with Tailwind CSS. Renders chat, project dashboard, research lab, skills library, file browser, git panel, and news dashboard.
2. **Backend** -- Node.js Express server with WebSocket support. Manages projects, sessions, authentication, skills, auto-research orchestration, and agent lifecycle.
3. **Agent Integration Layer** -- Individual modules that wrap each AI provider's SDK or CLI, translating between the unified Dr. Claw protocol and provider-specific APIs.

### 1.3 Key Design Decisions

- **Multi-provider architecture**: Rather than being tied to one LLM, Dr. Claw abstracts the agent interface so users can switch between Claude, Gemini, Codex, OpenRouter (any model), and local GPU models.
- **Skills as markdown**: Research procedures are encoded as `SKILL.md` files (YAML frontmatter + markdown body). Agents discover and read these at runtime -- no compiled plugin system.
- **Pipeline-as-data**: Research state is stored in JSON files (`.pipeline/docs/research_brief.json`, `.pipeline/tasks/tasks.json`) in the project directory, making it portable and git-trackable.
- **WebSocket streaming**: All agent interactions stream through WebSockets in real time, allowing the UI to show incremental tool use, thinking, and outputs.
- **SQLite for metadata**: A single SQLite database (`auth.db`) stores users, sessions, projects, references, tags, auto-research runs, and app settings. Project files remain on the filesystem.

---

## 2. Repository Structure

```
dr-claw/
|-- agent-harness/          # Python CLI harness ("drclaw" command)
|   |-- cli_anything/drclaw/  # Python package with Click CLI
|   |   |-- core/             # Core modules: chat, projects, sessions, taskmaster
|   |   |-- utils/            # Output helpers, OpenClaw bridges
|   |   +-- tests/
|   |-- DRCLAW.md             # Agent instructions for the harness
|   |-- SKILL.md              # Skill definition for agent-harness
|   +-- setup.py              # Python package definition
|
|-- community-tools/         # Third-party tool integrations
|   |-- autoresearch/          # Karpathy's autoresearch integration
|   +-- autoresearchclaw/      # AutoResearchClaw wrapper
|
|-- docs/                    # Documentation
|   |-- configuration.md      # Full environment reference
|   |-- faq.md                # Troubleshooting
|   |-- pipeline-outputs.md   # Artifact structure documentation
|   |-- skills-taxonomy-v2.md # Skills classification schema
|   +-- plans/                # Design documents
|
|-- electron/                # Desktop app (Electron wrapper)
|   |-- main.mjs              # Electron main process
|   |-- preload.mjs           # IPC bridge
|   +-- cli.mjs               # Build/dev CLI for desktop
|
|-- public/                  # Static assets (icons, screenshots, manifest)
|
|-- scripts/                 # Build and utility scripts
|   |-- check-deps.js         # Dependency validator (predev hook)
|   |-- native-runtime.mjs    # native module (node-pty) resolution
|   +-- export-skills-catalog-v2.mjs  # Skills catalog generator
|
|-- server/                  # Backend (Express + WebSocket server)
|   |-- index.js              # Main server entry point (~1400 lines)
|   |-- cli.js                # CLI entry point (dr-claw command)
|   |-- cli-chat.js           # Terminal chat via OpenRouter
|   |-- claude-sdk.js         # Claude Code SDK integration
|   |-- gemini-cli.js         # Gemini CLI process spawning
|   |-- openai-codex.js       # OpenAI Codex SDK integration
|   |-- openrouter.js         # OpenRouter API integration
|   |-- local-gpu.js          # Local GPU model integration
|   |-- nano-claude-code.js   # Nano Claude Code integration
|   |-- projects.js           # Project discovery and management (~2000 lines)
|   |-- compute-node.js       # Remote compute node management
|   |-- mcp-compute.js        # MCP server for compute tools
|   |-- telemetry.js          # Usage telemetry
|   |-- database/
|   |   |-- db.js             # SQLite database operations (~1950 lines)
|   |   +-- init.sql           # Schema definition
|   |-- constants/
|   |   +-- config.js          # Platform mode constants
|   |-- middleware/
|   |   +-- auth.js            # JWT + API key authentication
|   |-- routes/                # Express route modules (20 files)
|   |   |-- agent.js           # External agent API
|   |   |-- auto-research.js   # Auto research orchestration (~560 lines)
|   |   |-- projects.js        # Workspace creation/management
|   |   |-- skills.js          # Skills CRUD and import (~950 lines)
|   |   |-- news.js            # News dashboard feeds
|   |   |-- references.js      # Literature library (Zotero)
|   |   |-- settings.js        # User/app settings
|   |   |-- taskmaster.js      # Pipeline task management
|   |   |-- compute.js         # Compute node management
|   |   +-- ...                # git, mcp, auth, user, telemetry, etc.
|   |-- templates/             # Agent instruction templates
|   |   |-- index.js           # Template writer
|   |   |-- CLAUDE.md          # Claude Code system prompt template
|   |   |-- AGENTS.md          # Generic agent instructions
|   |   |-- GEMINI.md          # Gemini-specific instructions
|   |   +-- cursor-project.md  # Cursor rules template
|   |-- utils/                 # Server utilities (20+ files)
|   |   |-- skillExpander.js   # /skill-name -> SKILL.md resolution
|   |   |-- memoryPrompt.js    # User memory injection
|   |   |-- computeGuardPrompt.js  # GPU availability guard
|   |   |-- sessionIndex.js    # Session metadata indexing
|   |   |-- permissions.js     # Tool permission management
|   |   |-- zotero-client.js   # Zotero API client
|   |   +-- ...                # btw, codex, gemini, git, mcp helpers
|   +-- taskmaster-templates/  # Research project templates (JSON)
|       |-- ai-research-method-model.json
|       |-- ai-research-dataset.json
|       +-- ai-research-position-paper.json
|
|-- shared/                  # Code shared between server and frontend
|   |-- modelConstants.js     # Model definitions for all providers
|   |-- errorClassifier.js    # Error categorization
|   |-- geminiThinkingSupport.js  # Gemini thinking mode config
|   +-- geminiThoughtParser.js    # Gemini thought block parsing
|
|-- skills/                  # 114 research skill directories
|   |-- aris-*/               # ARIS pipeline skills (30+ skills)
|   |-- ds-*/                 # DeepScientist skills (12 skills)
|   |-- inno-*/               # Innovation/research core skills (15 skills)
|   |-- autoresearch/         # Karpathy-inspired autonomous iteration
|   |-- skill-tag-mapping.json      # Skill metadata overrides
|   |-- stage-skill-map.json        # Stage-to-skill mapping
|   |-- skills-catalog-v2.json      # Full skill catalog
|   +-- skills-taxonomy-v2.schema.json  # Taxonomy schema
|
|-- src/                     # Frontend (React) source
|   |-- App.tsx               # Root component with routing
|   |-- main.jsx              # React entry point
|   |-- components/           # UI components (~80 files)
|   |   |-- app/              # AppContent layout
|   |   |-- chat/             # Chat interface
|   |   |-- sidebar/          # Navigation sidebar
|   |   |-- project-dashboard/  # Project overview
|   |   |-- compute-dashboard/  # Compute management
|   |   |-- news-dashboard/   # News feeds
|   |   |-- survey/           # Literature survey views
|   |   |-- settings/         # Settings panels
|   |   |-- references/       # Reference library UI
|   |   +-- ui/               # Shared UI primitives
|   |-- contexts/             # React contexts
|   |   |-- AuthContext.jsx   # Authentication state
|   |   |-- WebSocketContext.tsx  # WebSocket connection
|   |   |-- TaskMasterContext.jsx  # Pipeline task state
|   |   +-- ThemeContext.jsx  # Dark/light theme
|   |-- hooks/                # Custom React hooks
|   |-- i18n/                 # Internationalization (en, zh-CN)
|   |-- constants/            # Frontend constants
|   |-- types/                # TypeScript type definitions
|   +-- utils/                # Frontend utilities
|
|-- test/                    # Test suite (Vitest + Playwright)
|-- package.json             # Dependencies and scripts
|-- vite.config.js           # Vite build configuration
|-- tailwind.config.js       # Tailwind CSS configuration
+-- tsconfig.json            # TypeScript configuration
```

---

## 3. Pipeline Stages

Dr. Claw organizes research into five sequential stages. Each stage has a dedicated directory in the project workspace and one or more skills that produce its artifacts.

### 3.1 Stage 1: Survey

**Purpose:** Literature review, code survey, and resource gathering.

**Key skills:** `inno-prepare-resources`, `inno-code-survey`, `inno-deep-research`, `paper-analyzer`, `paper-finder`, `aris-research-lit`, `aris-semantic-scholar`, `aris-arxiv`

**Outputs:**
- `Survey/reports/` -- Literature review summaries with citations
- `Survey/references/` -- Downloaded papers and structured notes

**How it works:** The agent reads the research brief from `.pipeline/docs/research_brief.json`, searches arXiv, Semantic Scholar, and the web for relevant papers, downloads and analyzes them, then produces structured survey reports. The `inno-deep-research` skill uses multi-round web search with progressive refinement.

### 3.2 Stage 2: Ideation

**Purpose:** Generate and evaluate research ideas.

**Key skills:** `inno-idea-generation`, `inno-idea-eval`, `aris-idea-creator`, `aris-idea-discovery`, `aris-novelty-check`

**Outputs:**
- `Ideation/ideas/` -- Structured brainstorming outputs with multi-persona evaluation scores
- `Ideation/references/` -- Supporting materials

**How it works:** Ideas are generated using creative frameworks (SCAMPER, SWOT, Mind Mapping). Each idea is evaluated by multiple simulated expert personas who score dimensions like novelty, feasibility, and impact. The ARIS variant (`aris-idea-discovery`) chains literature survey, idea creation, novelty checking, and preliminary review into an automated pipeline.

### 3.3 Stage 3: Experiment

**Purpose:** Implement experiments, run them, and analyze results.

**Key skills:** `inno-experiment-dev`, `inno-experiment-analysis`, `aris-experiment-plan`, `aris-run-experiment`, `aris-monitor-experiment`, `aris-analyze-results`

**Outputs:**
- `Experiment/core_code/` -- Implementation code
- `Experiment/datasets/` -- Downloaded or generated datasets
- `Experiment/code_references/` -- Cloned repos, architecture maps
- `Experiment/analysis/` -- Statistical analysis, tables, paper-ready figures

**How it works:** The agent implements experiments from the plan, optionally deploying to remote GPU servers via the compute node system. `aris-run-experiment` can SSH into configured servers, sync code, and launch training in screen sessions. Results are analyzed with statistical methods and turned into publication-ready figures.

**Compute Guard:** Before running any GPU-intensive task, the system prompt includes a mandatory `COMPUTE_GUARD_BLOCK` (defined in `server/utils/computeGuardPrompt.js`) that forces the agent to check GPU availability via `nvidia-smi` and refuse to fabricate results if compute is unavailable.

### 3.4 Stage 4: Publication

**Purpose:** Write the academic paper, generate figures, and review.

**Key skills:** `inno-paper-writing`, `inno-figure-gen`, `inno-paper-reviewer`, `inno-humanizer`, `inno-reference-audit`, `inno-rclone-to-overleaf`, `aris-paper-write`, `aris-auto-review-loop`

**Outputs:**
- `Publication/paper/` -- Academic manuscript in IEEE/ACM format with LaTeX math

**How it works:** The agent produces a structured manuscript following academic conventions. The `aris-auto-review-loop` implements cross-model adversarial review: GPT reviews Claude's paper (or vice versa), providing scores and weaknesses. Claude implements fixes, and the cycle repeats up to 4 rounds until the score exceeds 6/10.

### 3.5 Stage 5: Promotion

**Purpose:** Create presentation materials and project homepage.

**Key skills:** `making-academic-presentations`, `aris-paper-slides`, `aris-paper-poster`

**Outputs:**
- `Promotion/slides/` -- Presentation deck
- `Promotion/audio/` -- TTS narration
- `Promotion/video/` -- Demo video
- `Promotion/homepage/` -- Project landing page

### 3.6 Pipeline Initialization

When a new project is created, the `inno-pipeline-planner` skill is invoked. It interactively collects the user's research topic, target venue, core question, and methods, then generates:

- `.pipeline/docs/research_brief.json` -- Defines stage goals, quality gates, recommended skills, and the `startStage` (which stage to begin from)
- `.pipeline/tasks/tasks.json` -- A task list where each task has `id`, `title`, `description`, `status`, `stage`, `priority`, `dependencies`, `suggestedSkills`, and `nextActionPrompt`

The `nextActionPrompt` field is the literal prompt sent to the agent when running the task, either manually via "Use in Chat" or automatically via Auto Research.

---

## 4. Core Components

### 4.1 Agent Integration Layer

Each AI provider has a dedicated server-side module that implements a common interface:

| Module | File | Provider | Integration Method |
|--------|------|----------|--------------------|
| Claude Code | `server/claude-sdk.js` | Anthropic | `@anthropic-ai/claude-agent-sdk` (direct SDK) |
| Gemini CLI | `server/gemini-cli.js` | Google | Process spawning via `node-pty` |
| Codex | `server/openai-codex.js` | OpenAI | `@openai/codex-sdk` (direct SDK) |
| OpenRouter | `server/openrouter.js` | Any model | OpenAI-compatible chat completions API |
| Local GPU | `server/local-gpu.js` | Self-hosted | Custom integration |
| Nano Claude Code | `server/nano-claude-code.js` | Anthropic | External Python agent |

**Common interface per provider:**

```javascript
queryXxx(command, options, ws)      // Execute a prompt, stream results over WebSocket
abortXxxSession(sessionId)          // Cancel an active session
isXxxSessionActive(sessionId)       // Check if running
getXxxSessionStartTime(sessionId)   // Get start timestamp
getActiveXxxSessions()              // List all active sessions
```

#### 4.1.1 Claude SDK Integration (`server/claude-sdk.js`)

The primary integration, using `@anthropic-ai/claude-agent-sdk`. Key flow:

1. `mapCliOptionsToSDK()` (line 86) translates Dr. Claw options to SDK format, including permission mode, allowed/disallowed tools, model selection, and system prompt injection.
2. User memory and compute guard are appended to the system prompt via `buildMemoryBlock()` and `COMPUTE_GUARD_BLOCK`.
3. MCP server configurations are loaded from `~/.claude.json` (line 400).
4. An active compute node (if configured) is injected as an MCP server (line 530).
5. The `canUseTool` callback (line 566) implements tool permission checking: interactive tools like `AskUserQuestion` always prompt the user; other tools check allowed/disallowed lists, then fall back to asking via WebSocket.
6. The `query()` function returns an async iterator that streams `assistant`, `result`, and other message types. Each message is transformed and forwarded to the WebSocket.
7. Token budget tracking uses `extractTokenBudgetFromUsage()` (line 283) to report context window utilization.
8. Session IDs are captured from the first message and tracked in `activeSessions` Map.

#### 4.1.2 OpenRouter Integration (`server/openrouter.js`)

A fully custom agent implementation using OpenRouter's OpenAI-compatible API with function calling:

1. Defines 10 tool schemas in OpenAI function-calling format (line 43): `Read`, `Write`, `Edit`, `Bash`, `Glob`, `Grep`, `LS`, `WebFetch`, `WebSearch`, `TodoWrite`.
2. Each tool is implemented locally in `executeTool()` (line 227) -- file I/O, shell commands, web search via DuckDuckGo HTML scraping.
3. Uses an agentic loop (line 689) that runs up to `MAX_AGENT_TURNS` (30) iterations, calling the model, executing tool calls, and feeding results back.
4. Sessions are persisted as JSONL files in `~/.dr-claw/openrouter-sessions/`.
5. The system prompt is built dynamically from `AGENTS.md`, `instance.json`, and the first 10 skills in `.agents/skills/`.
6. Skill slash commands (e.g., `/autoresearch`) are expanded into full SKILL.md content by `expandSkillCommand()` before sending to the model.

#### 4.1.3 Gemini CLI Integration (`server/gemini-cli.js`)

Spawns the Gemini CLI as a child process via `node-pty` and parses its streaming JSON output. Maps Gemini tool names to canonical names (e.g., `run_shell_command` -> `Bash`, `google_web_search` -> `WebSearch`). Implements tool permission checking that mirrors the Claude pattern.

#### 4.1.4 Codex SDK Integration (`server/openai-codex.js`)

Uses `@openai/codex-sdk` directly. Transforms Codex events (agent messages, tool calls, patches) into the unified WebSocket message format. Handles `exec_command` and `patch_apply` events from the Codex agent.

### 4.2 Skills System

Skills are the core abstraction for research procedures. Each skill is a directory under `skills/` containing at minimum a `SKILL.md` file.

#### 4.2.1 Skill Structure

```
skills/
  inno-paper-writing/
    SKILL.md              # Main procedure (YAML frontmatter + markdown)
    prompts/              # Optional prompt templates
    references/           # Optional reference materials
```

The YAML frontmatter in `SKILL.md` defines metadata:

```yaml
---
name: inno-paper-writing
description: Write an academic paper following the IMRAD structure
version: 1.0.0
license: MIT
metadata:
  author: OpenLAIR
---
```

#### 4.2.2 Skill Discovery and Linking

When a project is created or a session starts, `ensureProjectSkillLinks()` in `server/projects.js` symlinks all skills from the global `skills/` directory into the project's `.claude/skills/`, `.agents/skills/`, and other provider-specific directories. This allows every agent to discover skills by reading their working directory.

#### 4.2.3 Skill Expansion

For non-Claude providers (OpenRouter, Gemini, Codex) that cannot natively interpret `/skill-name` slash commands, the `expandSkillCommand()` function in `server/utils/skillExpander.js` resolves the command:

1. Parses `/skill-name arguments` from the user message (line 114)
2. Searches for `SKILL.md` in multiple locations (line 18): `.agents/skills/`, `.claude/skills/`, `.gemini/skills/`, global `skills/`, and `~/.claude/skills/`
3. Reads the SKILL.md content and builds a sub-skill index for any nested `/skill-name` references
4. Returns the full skill content prepended with "Follow the procedure below exactly"

#### 4.2.4 Skill Categories (114 skills total)

| Category | Prefix | Count | Purpose |
|----------|--------|-------|---------|
| ARIS | `aris-*` | ~35 | End-to-end research pipeline, cross-model review |
| DeepScientist | `ds-*` | ~12 | 13-stage research OS (ICLR 2026) |
| Innovation Core | `inno-*` | ~15 | Pipeline planner, paper writing, experiments |
| Autoresearch | `autoresearch` | 1 | Karpathy-inspired autonomous iteration (9 subcommands) |
| Domain-specific | various | ~20 | Bioinformatics, chemistry, ML specialties |
| Infrastructure | various | ~15 | Training, deployment, MLOps |
| General | various | ~16 | Academic research, scientific writing |

#### 4.2.5 Skill Metadata Management

Two JSON files in `skills/` track metadata:

- `skill-tag-mapping.json` -- Stage and domain override tags per skill
- `stage-skill-map.json` -- Maps skills to pipeline stages and tracks origin (built-in, downloaded, user-import, local-import)

The `skills-catalog-v2.json` contains a structured catalog with full taxonomy data.

### 4.3 Multi-Agent Orchestration

#### 4.3.1 Auto Research (`server/routes/auto-research.js`)

The Auto Research system executes the full task pipeline autonomously:

1. **Status check** (`GET /:projectName/status`, line 411): Reads `.pipeline/tasks/tasks.json`, determines eligible tasks, and checks for active runs.
2. **Start** (`POST /:projectName/start`, line 453): Creates a run record in the database, then spawns `runAutoResearch()`.
3. **Execution loop** (`runAutoResearch()`, line 305):
   - Reads pipeline state from the filesystem
   - Finds the next actionable task (in-progress or pending)
   - Extracts the `nextActionPrompt` from the task
   - Routes to the correct provider (`queryClaudeSDK`, `queryCodex`, `spawnGemini`, or `queryOpenRouter`)
   - Wraps execution with a timeout (`TASK_TIMEOUT_MS`, default 30 minutes)
   - After execution, re-reads pipeline state to verify the task transitioned to "done"
   - Moves to the next task, repeating until all tasks complete
4. **Completion**: Sends an email notification via `sendAutoResearchCompletionEmail()` and updates the database.
5. **Stale run recovery** (`reconcileActiveRun()`, line 285): If a run's session is no longer active and has no runtime state, it is automatically marked as failed or cancelled.

The `AutoResearchWriter` class (line 212) is a fake WebSocket that captures the `session-created` event to track the agent session ID.

#### 4.3.2 Cross-Model Adversarial Review

The `aris-auto-review-loop` skill implements a unique multi-model review cycle:
- Claude writes/revises the paper
- GPT (via OpenRouter or Codex MCP) reviews the paper, providing scores and weaknesses
- Claude implements the fixes
- The cycle repeats up to 4 rounds

The `REVIEWER_DIFFICULTY` constant controls how adversarial the reviewer is (medium, hard, nightmare).

### 4.4 Memory and State Management

#### 4.4.1 Project State (Filesystem)

Each project stores its state in the filesystem:

- `instance.json` -- Root config with absolute paths to all pipeline directories
- `.pipeline/docs/research_brief.json` -- Research brief with goals, methods, quality gates
- `.pipeline/tasks/tasks.json` -- Task list with status tracking
- `.pipeline/config.json` -- Pipeline configuration metadata

#### 4.4.2 Session State (Database + Filesystem)

Sessions are tracked in two places:

1. **SQLite** (`session_metadata` table in `server/database/db.js`): Stores session ID, project name, provider, display name, last activity, message count, metadata (JSON), and tags.
2. **Provider-specific storage**:
   - Claude: `~/.claude/projects/{encoded-path}/{session-id}.jsonl`
   - Codex: `~/.codex/sessions/{session-id}.jsonl`
   - Gemini: `~/.gemini/sessions/{session-id}/`
   - OpenRouter: `~/.dr-claw/openrouter-sessions/{session-id}.jsonl`

#### 4.4.3 User Memory (`server/utils/memoryPrompt.js`)

Users can save persistent memories (up to 50, max 500 chars each) in the `user_memories` table. When a session starts, `buildMemoryBlock()` (line 20) reads enabled memories and injects them into the system prompt:

```
# User Memories
The following are things the user has asked you to remember:
- Always use PyTorch over TensorFlow
- My research focus is NLP for low-resource languages
```

#### 4.4.4 Session Stage Tags

Sessions are auto-tagged with pipeline stage labels (Survey, Ideation, Experiment, Publication, Promotion) stored in `project_tags` and `session_tag_links` tables. Tags can be set automatically by the agent context or manually by the user. The `tagDb` module in `server/database/db.js` (line 1211) manages the full tag lifecycle.

### 4.5 Review and Verification Mechanisms

1. **Compute Guard**: System prompt injection that forces agents to verify GPU availability before running experiments (`server/utils/computeGuardPrompt.js`).
2. **Tool Permissions**: The `canUseTool` callback in `claude-sdk.js` and `checkToolPermission()` in `openrouter.js` implement a three-tier permission system:
   - Bypass mode: All tools allowed
   - Accept edits mode: Read + write tools allowed
   - Plan mode: Only read-only tools allowed
   - Default: User prompted via WebSocket for unrecognized tools
3. **Reference Audit**: The `inno-reference-audit` skill verifies that all citations in a paper are real and correctly formatted.
4. **Auto Review Loop**: Cross-model adversarial paper review (see section 4.3.2).

---

## 5. Tech Stack

### 5.1 Backend

- **Runtime**: Node.js 20/22/24 (see `.nvmrc`: `22`)
- **HTTP Server**: Express 4.18
- **WebSocket**: `ws` 8.14 -- used for real-time agent streaming, project updates, and terminal sessions
- **Database**: better-sqlite3 12.2 (synchronous SQLite driver)
- **Terminal Emulation**: `node-pty` 1.1 -- provides pseudo-terminal for embedded shell and Gemini CLI spawning
- **File Watching**: `chokidar` 4.0 -- watches `~/.claude/projects/`, `~/.codex/sessions/`, etc. for real-time project updates

### 5.2 Frontend

- **Framework**: React 18 with `react-router-dom` 6
- **Build Tool**: Vite 7
- **Styling**: Tailwind CSS 3.4 with `@tailwindcss/typography`
- **Code Editor**: CodeMirror (`@uiw/react-codemirror`) with multi-language support
- **Terminal**: xterm.js (`@xterm/xterm` 5.5) with WebGL renderer
- **Markdown**: `react-markdown` with `remark-gfm`, `remark-math`, `rehype-katex` for LaTeX rendering
- **Diagrams**: Mermaid 11 for flowcharts and sequence diagrams
- **Internationalization**: `i18next` + `react-i18next` (English and Chinese)
- **Search**: Fuse.js 7 for fuzzy matching

### 5.3 Desktop App

- **Framework**: Electron 37
- **Builder**: `electron-builder` 26
- **Entry**: `electron/main.mjs` spawns the Express server and opens a BrowserWindow
- **IPC**: `electron/preload.mjs` exposes a bridge for desktop-specific features

### 5.4 Agent SDKs

- `@anthropic-ai/claude-agent-sdk` 0.1.71
- `@openai/codex-sdk` 0.125
- `@modelcontextprotocol/sdk` 1.29 (MCP protocol support)
- `@octokit/rest` 22 (GitHub API)

### 5.5 Python CLI

- The `agent-harness/` contains a Python package using Click for CLI argument parsing and `requests` for HTTP communication with the server.

---

## 6. Configuration

### 6.1 Environment Variables (`.env`)

The primary configuration file. Key variables defined in `.env.example`:

| Variable | Default | Purpose |
|----------|---------|---------|
| `PORT` | `3001` | Backend server port |
| `VITE_PORT` | `5173` | Frontend dev server port |
| `HOST` | `0.0.0.0` | Bind address |
| `JWT_SECRET` | (unset) | JWT signing key -- **must set** for non-localhost |
| `DATABASE_PATH` | `server/database/auth.db` | SQLite database location |
| `WORKSPACES_ROOT` | `~/dr-claw` | Default project storage root |
| `OPENROUTER_API_KEY` | (unset) | OpenRouter API key |
| `OPENROUTER_MODEL` | `anthropic/claude-sonnet-4` | Default OpenRouter model |
| `CONTEXT_WINDOW` | auto-detected | Override context window size |
| `CLAUDE_CLI_PATH` | `claude` | Path to Claude CLI binary |
| `GEMINI_CLI_PATH` | `gemini` | Path to Gemini CLI binary |
| `CODEX_CLI_PATH` | `codex` | Path to Codex CLI binary |
| `AUTO_RESEARCH_TASK_TIMEOUT_MS` | `1800000` (30 min) | Auto research task timeout |

### 6.2 Project Configuration

- `~/.claude/project-config.json` -- Stores manually added projects, workspace root override, and deleted project metadata.
- `instance.json` (per project) -- Absolute paths for all pipeline directories, research topic, and project metadata.
- `.pipeline/config.json` -- Pipeline-level config (e.g., `intakeCompleted` flag).

### 6.3 Agent Instruction Templates

When a project is initialized, `writeProjectTemplates()` in `server/templates/index.js` writes four template files (if they do not already exist):

1. `CLAUDE.md` -- System prompt for Claude Code sessions. Defines the research assistant role, pipeline workflow, intake procedure, skill discovery, and project rules.
2. `AGENTS.md` -- Generic agent instructions (used by OpenRouter/Gemini).
3. `GEMINI.md` -- Gemini-specific instructions.
4. `.cursor/rules/project.md` -- Cursor agent rules.

Templates support `{{SNIPPET_NAME}}` placeholders that are resolved from `_snippet-name-snippet.md` files at write time (e.g., `{{COMPUTE-GUARD}}` injects the compute guard block).

### 6.4 Skills Configuration

- `skills/skill-tag-mapping.json` -- Maps skill names to stage and domain tags
- `skills/stage-skill-map.json` -- Tracks skill origins (built-in, downloaded, user-import)
- `skills/skills-catalog-v2.json` -- Full structured catalog

---

## 7. Data Flow

### 7.1 End-to-End Research Pipeline Flow

```
User describes idea in Chat
         |
         v
Agent runs inno-pipeline-planner skill
         |
         v
Generates research_brief.json + tasks.json
         |
         v
User clicks "Auto Research" or "Use in Chat"
         |
    +----+----+
    |         |
    v         v
Auto Research   Manual Chat
(sequential     (one task
 task loop)      at a time)
    |              |
    +------+-------+
           |
           v
Agent executes task via provider SDK/CLI
(Claude SDK, Gemini CLI, Codex SDK, OpenRouter API)
           |
           v
Agent reads SKILL.md, follows procedure
           |
           v
Agent uses tools (Read, Write, Bash, WebSearch, etc.)
           |
           v
Results written to pipeline directories
(Survey/, Ideation/, Experiment/, Publication/, Promotion/)
           |
           v
Agent updates tasks.json (status -> done)
           |
           v
Next task begins (loop continues)
           |
           v
Completion email sent
```

### 7.2 WebSocket Message Flow

```
Frontend                    Backend                     Agent SDK
   |                          |                            |
   |-- WS connect ---------->|                            |
   |   (Bearer token auth)   |                            |
   |                          |                            |
   |-- chat message --------->|                            |
   |   {provider, prompt,     |                            |
   |    sessionId, options}   |                            |
   |                          |-- queryClaudeSDK() ------->|
   |                          |   (prompt, options, ws)    |
   |                          |                            |
   |                          |<- session_id --------------|
   |<- session-created -------|                            |
   |                          |                            |
   |                          |<- assistant message -------|
   |<- claude-response -------|   (streaming)              |
   |   (tool_use, text)       |                            |
   |                          |                            |
   |                          |<- tool permission req -----|
   |<- permission-request ----|                            |
   |-- permission-response -->|                            |
   |                          |-- resolve approval ------->|
   |                          |                            |
   |                          |<- result ------------------|
   |<- claude-complete -------|                            |
   |<- token-budget ----------|                            |
```

### 7.3 Project Discovery Flow

The `getProjects()` function in `server/projects.js` discovers projects from multiple sources:

1. **Database** (`projects` table): Manually added or previously discovered projects
2. **Claude projects** (`~/.claude/projects/`): Scans directory entries, extracts project paths from `.jsonl` session files
3. **Codex sessions** (`~/.codex/sessions/`): Matches session directories to known projects
4. **Gemini sessions** (`~/.gemini/sessions/`): Matches session files to known projects
5. **OpenRouter sessions** (`~/.dr-claw/openrouter-sessions/`): Matches session files

File system watchers (`chokidar`) monitor all provider directories and broadcast `projects_updated` WebSocket events on changes. A debounce timer (`WATCHER_DEBOUNCE_MS` = 1000ms) prevents excessive updates.

---

## 8. Key Abstractions

### 8.1 Session

A session represents a single conversation thread with an AI agent. Sessions are identified by a UUID and are scoped to a project and provider. The `sessionDb` in `server/database/db.js` manages session metadata with methods like `upsertSession()`, `upsertSessionFromSource()`, `migrateSessionId()`, `updateSessionMetadata()`, and `updateSessionContextReview()`.

Sessions have modes:
- `research` (default) -- Full research assistant behavior
- `workspace_qa` -- Lightweight Q&A mode that skips pipeline initialization

### 8.2 Project

A project maps to a filesystem directory and contains pipeline state, agent configuration, and research artifacts. Projects are identified by an encoded path (slashes replaced with dashes). The `projectDb` in `server/database/db.js` manages project records with `upsertProject()`, `getProjectByPath()`, `migrateProjectIdentity()`.

### 8.3 Pipeline Task

Tasks are the atomic units of the research pipeline. Each task in `.pipeline/tasks/tasks.json` has:

```json
{
  "id": 1,
  "title": "Literature Survey on Transformer Efficiency",
  "description": "...",
  "status": "pending",         // pending | in-progress | done | review | deferred | cancelled
  "stage": "survey",           // survey | ideation | experiment | publication | promotion
  "priority": "high",
  "dependencies": [],
  "taskType": "survey",
  "inputsNeeded": ["research topic"],
  "suggestedSkills": ["inno-deep-research", "paper-finder"],
  "nextActionPrompt": "Read the skill SKILL.md and follow it to survey..."
}
```

### 8.4 Research Brief

The research brief (`.pipeline/docs/research_brief.json`) is the control document for the entire pipeline. It defines:
- Research topic and goals
- Target venue and paper type
- Stage definitions with goals, required elements, and quality gates
- `pipeline.startStage` -- which stage to begin from (e.g., skip survey if user already has results)
- Skill recommendations per stage

### 8.5 Skill

A skill is a directory containing `SKILL.md` with YAML frontmatter and procedural markdown. Skills are discovered by scanning provider-specific directories in the project workspace. They are the primary mechanism for encoding research methodology as structured prompts.

### 8.6 AutoResearchWriter

A lightweight WebSocket-like writer class (`server/routes/auto-research.js`, line 212) used during auto research to capture session events without a real WebSocket connection. It implements `send()` and `setSessionId()` to match the interface expected by agent query functions.

### 8.7 Compute Node

Remote compute resources managed via `server/compute-node.js`. Nodes are stored in `~/.openclaw/compute-node.json` with SSH connection details. When active, compute tools are injected as an MCP server into agent sessions, providing `compute_run`, `compute_sync`, `compute_slurm_submit`, and other remote execution commands.

---

## 9. Dependencies

### 9.1 Runtime Dependencies (Key)

| Package | Version | Purpose |
|---------|---------|---------|
| `express` | 4.18 | HTTP server |
| `ws` | 8.14 | WebSocket server |
| `better-sqlite3` | 12.2 | SQLite database (synchronous) |
| `node-pty` | 1.1-beta34 | Pseudo-terminal for shell/CLI spawning |
| `chokidar` | 4.0 | Filesystem watching for project discovery |
| `@anthropic-ai/claude-agent-sdk` | 0.1.71 | Claude Code SDK |
| `@openai/codex-sdk` | 0.125 | OpenAI Codex SDK |
| `@modelcontextprotocol/sdk` | 1.29 | MCP protocol support |
| `react` | 18.2 | UI framework |
| `react-router-dom` | 6.8 | Client-side routing |
| `@uiw/react-codemirror` | 4.23 | Code editor |
| `@xterm/xterm` | 5.5 | Terminal emulator |
| `react-markdown` | 10.1 | Markdown rendering |
| `mermaid` | 11.12 | Diagram rendering |
| `katex` | 0.16 | LaTeX math rendering |
| `fuse.js` | 7.0 | Fuzzy search |
| `i18next` | 25.7 | Internationalization |
| `multer` | 2.0 | File upload handling |
| `jsonwebtoken` | 9.0 | JWT authentication |
| `bcryptjs` | 3.0 | Password hashing |
| `gray-matter` | 4.0 | YAML frontmatter parsing |
| `adm-zip` | 0.5 | Zip file handling (skill uploads) |
| `@octokit/rest` | 22.0 | GitHub API (repo cloning) |
| `node-fetch` | 2.7 | HTTP client |

### 9.2 Dev Dependencies (Key)

| Package | Version | Purpose |
|---------|---------|---------|
| `vite` | 7.0 | Build tool and dev server |
| `vitest` | 4.1 | Unit testing |
| `@playwright/test` | 1.58 | E2E testing |
| `electron` | 37.3 | Desktop app shell |
| `electron-builder` | 26.0 | Desktop app packaging |
| `tailwindcss` | 3.4 | CSS framework |
| `typescript` | 5.9 | Type checking |

---

## 10. Entry Points

### 10.1 CLI Commands

**`dr-claw` / `vibelab`** (defined in `server/cli.js`, registered in `package.json` bin):

| Command | Description |
|---------|-------------|
| `dr-claw` or `dr-claw start` | Start the Express server (default) |
| `dr-claw chat --model <slug>` | Interactive terminal chat via OpenRouter |
| `dr-claw status` | Show configuration and data locations |
| `dr-claw update` | Update to latest npm version |
| `dr-claw help` | Show help |
| `dr-claw version` | Show version |

**`drclaw`** (Python CLI, defined in `agent-harness/cli_anything/drclaw/drclaw_cli.py`):

| Command | Description |
|---------|-------------|
| `drclaw auth login/logout/status` | Authentication |
| `drclaw projects list/add/rename/delete` | Project management |
| `drclaw sessions list/messages` | Session access |
| `drclaw taskmaster status/tasks/next` | Pipeline task management |
| `drclaw chat send/reply/waiting` | Chat and session control |
| `drclaw digest project/daily` | Progress digests |
| `drclaw openclaw install/configure/push` | OpenClaw integration |
| `drclaw skills list` | Skills listing |

### 10.2 npm Scripts

| Script | Description |
|--------|-------------|
| `npm run dev` | Start both server and Vite dev server with hot reload |
| `npm run server` | Start Express server only (with file watching for auto-restart) |
| `npm run client` | Start Vite dev server only |
| `npm run build` | Production build |
| `npm start` | Build + start server |
| `npm test` | Run vitest |
| `npm run desktop:dev` | Launch Electron in dev mode |
| `npm run desktop:dist` | Build desktop installer |

### 10.3 API Endpoints

All API routes are under `/api/` and require JWT authentication (except `/api/auth/*`):

| Route | Module | Purpose |
|-------|--------|---------|
| `/api/auth/*` | `routes/auth.js` | Registration, login, JWT refresh |
| `/api/projects/*` | `routes/projects.js` | Workspace creation, cloning |
| `/api/auto-research/:project/*` | `routes/auto-research.js` | Auto research start/cancel/status |
| `/api/skills/*` | `routes/skills.js` | Skills CRUD, import, upload |
| `/api/taskmaster/*` | `routes/taskmaster.js` | Pipeline task management |
| `/api/references/*` | `routes/references.js` | Literature library (Zotero sync) |
| `/api/news/*` | `routes/news.js` | News dashboard feeds |
| `/api/settings/*` | `routes/settings.js` | User and app settings |
| `/api/compute/*` | `routes/compute.js` | Compute node management |
| `/api/git/*` | `routes/git.js` | Git operations |
| `/api/mcp/*` | `routes/mcp.js` | MCP server management |
| `/api/btw/*` | `routes/btw.js` | Side question (ephemeral) |
| `/api/agent/*` | `routes/agent.js` | External agent API |
| `/api/quick-qa/*` | `routes/quick-qa.js` | Workspace Q&A |
| `/api/user/*` | `routes/user.js` | User profile management |
| `/api/cli/*` | `routes/cli-auth.js` | CLI authentication |

### 10.4 WebSocket

A single WebSocket server on the same HTTP port handles:

1. **Chat sessions**: Agent streaming for all providers
2. **Terminal sessions**: PTY-backed interactive shell
3. **Project updates**: Real-time project/session change notifications

WebSocket messages use the `type` field to distinguish: `claude-response`, `openrouter-response`, `codex-response`, `gemini-response`, `session-created`, `claude-complete`, `claude-error`, `token-budget`, `projects_updated`, `terminal_data`, `claude-permission-request`, etc.

---

## 11. Extensibility

### 11.1 Adding a New Agent Provider

1. Create `server/new-provider.js` implementing the standard interface:
   - `queryNewProvider(command, options, ws)` -- streaming query
   - `abortNewProviderSession(sessionId)` -- cancellation
   - `isNewProviderSessionActive(sessionId)` -- status check
   - `getActiveNewProviderSessions()` -- listing
2. Import and register in `server/index.js` (follow the pattern of existing providers)
3. Add WebSocket message handling in the `wss.on('connection')` handler
4. Add model constants in `shared/modelConstants.js`
5. Add the provider to `normalizeAutoResearchProvider()` in `routes/auto-research.js` for auto research support
6. Create a frontend logo component and add the provider to the chat UI selector

### 11.2 Adding a New Skill

1. Create a directory under `skills/your-skill-name/`
2. Write `SKILL.md` with YAML frontmatter (`name`, `description`, `version`) and procedural markdown body
3. Optionally add `prompts/` and `references/` subdirectories
4. Update `skills/skill-tag-mapping.json` if the skill needs custom stage/domain tags
5. The skill will be automatically discovered and symlinked into projects on next session start

### 11.3 Adding a New Pipeline Stage

1. Add the stage to `PROJECT_PIPELINE_FOLDERS` in `server/projects.js` (line 92)
2. Add a default stage tag in `DEFAULT_STAGE_TAGS` in `server/database/db.js` (line 26)
3. Update the `CLAUDE.md` template to document the new stage
4. Create skills for the new stage
5. Update the `inno-pipeline-planner` skill to generate tasks for the new stage

### 11.4 Adding a New Research Template

1. Create a JSON file in `server/taskmaster-templates/` following the structure of `ai-research-method-model.json`
2. The template defines the research brief schema and pre-configured task pipeline

---

## 12. Relationship to Other Claw Projects

### 12.1 AutoResearchClaw

Referenced in `community-tools/autoresearchclaw/`. This appears to be a wrapper or integration point that connects the Dr. Claw workspace with a standalone AutoResearchClaw system. The community tools directory also contains `autoresearch/` which integrates Karpathy's autoresearch methodology.

### 12.2 OpenClaw

OpenClaw is a mobile/chat-based "secretary" layer that connects to Dr. Claw via the `drclaw` CLI. The integration is documented extensively in the README (see "OpenClaw Integration" section). Key components:

- `agent-harness/cli_anything/drclaw/core/openclaw_bridge.py` -- Sends messages through OpenClaw
- `agent-harness/cli_anything/drclaw/core/openclaw_daemon.py` -- Event-driven watcher daemon
- `server/utils/taskmaster-websocket.js` -- WebSocket event bridge for OpenClaw watcher

OpenClaw executes `drclaw --json` commands locally, receives structured `openclaw.*` schema payloads, and can push notifications to Feishu/Lark via a watcher process.

### 12.3 NeuroClaw

Not directly referenced in the codebase, though the compute node system (`server/compute-node.js`) and Slurm HPC integration suggest infrastructure that could support NeuroClaw-style neural computation workflows.

### 12.4 MetaClaw

Not directly referenced in the source code.

### 12.5 Claude Code Plugin (`dr-claw-plugin-cc`)

A separate repository (`OpenLAIR/dr-claw-plugin-cc`) that provides a lightweight version of Dr. Claw's research pipeline as a Claude Code plugin. It bundles ~60 skills and 3 project templates, using the same `research_brief.json` and `tasks.json` formats for interoperability.

---

## 13. Database Schema Summary

The SQLite database (`server/database/init.sql`) contains 10 tables:

| Table | Purpose |
|-------|---------|
| `users` | User accounts (username, password hash, git config, notification email, memory enabled) |
| `api_keys` | API keys for external access |
| `user_credentials` | Stored credentials (GitHub tokens, etc.) |
| `session_metadata` | Session index (ID, project, provider, display name, activity, tags metadata) |
| `project_tags` | Tag definitions per project (stage tags: survey, ideation, experiment, publication, promotion) |
| `session_tag_links` | Many-to-many: sessions to tags |
| `projects` | Project index (ID, path, user, starred, metadata) |
| `auto_research_runs` | Auto research execution records (status, progress, session ID) |
| `app_settings` | Key-value app configuration |
| `references_library` | Literature references (title, authors, DOI, abstract, Zotero sync) |
| `project_references` | Many-to-many: projects to references |
| `reference_tags` | Tags on references |
| `user_memories` | User memory items (content, category, enabled flag) |

---

## 14. Testing

- **Unit tests**: Vitest (`vitest.config.ts`), test files in `server/__tests__/` and `test/`
- **E2E tests**: Playwright (`playwright.config.ts`), spec files in `test/*.spec.ts`
- **Python tests**: pytest in `agent-harness/cli_anything/drclaw/tests/`
- **CI**: GitHub Actions workflows in `.github/workflows/` for CI, publishing, and desktop releases

---

## 15. Security Considerations

1. **Authentication**: JWT-based with bcrypt password hashing. API key authentication for programmatic access.
2. **Path validation**: `validateWorkspacePath()` in `routes/projects.js` prevents workspace creation in system directories (`/etc`, `/usr`, etc.) and enforces that paths stay within the user's home directory or configured workspace root.
3. **Tool permissions**: Three-tier system (bypass, accept edits, plan mode) with per-tool allow/deny lists and user confirmation via WebSocket.
4. **Path traversal protection**: `safePath()` in `server/utils/safePath.js` constrains all file operations to the project root.
5. **Compute guard**: System prompt injection prevents agents from fabricating experiment results when no GPU is available.
6. **Secret management**: Environment variables for API keys; credential storage in the database for GitHub tokens.
