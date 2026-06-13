# Claude Code Source - Buildable Research Fork

[![Build Status](https://img.shields.io/badge/build-passing-brightgreen)]()
[![Bun](https://img.shields.io/badge/runtime-Bun-f9f1e1?logo=bun)](https://bun.sh)
[![TypeScript](https://img.shields.io/badge/lang-TypeScript-3178c6?logo=typescript&logoColor=white)]()
[![Lines of Code](https://img.shields.io/badge/lines-512K+-blue)]()
[![License](https://img.shields.io/badge/license-research%20only-orange)]()
[![Stars](https://img.shields.io/github/stars/T-Lab-CUHKSZ/claude-code?style=social)](https://github.com/T-Lab-CUHKSZ/claude-code/stargazers)
[![Forks](https://img.shields.io/github/forks/T-Lab-CUHKSZ/claude-code?style=social)](https://github.com/T-Lab-CUHKSZ/claude-code/network/members)

> A **buildable, modifiable, and runnable** version of the Claude Code source.

Based on the Claude Code source snapshot publicly exposed on 2026-03-31 via an npm source map leak. The original snapshot contained only raw TypeScript source with no build configuration — it could not be compiled or run. This fork reconstructs the full build system and fixes all missing components to make it functional.

**[Quick Start](#quick-start)** | **[Architecture](#architecture-overview)** | **[Feature Flags](#feature-flags)** | **[Extension Guide](#extension-points-no-source-modification-needed)**

---

## Quick Start

### Prerequisites

- [Bun](https://bun.sh) >= 1.3.x
- Valid Anthropic authentication (OAuth via `claude login` or `ANTHROPIC_API_KEY`)

### Install & Run

```bash
git clone https://github.com/T-Lab-CUHKSZ/claude-code.git
cd claude-code

# Install dependencies (auto-creates bun:bundle polyfill via postinstall)
bun install

# Run directly
bun src/main.tsx -p "your prompt here" --output-format text
```

### Build (Optional)

```bash
# Compile to single bundle (~20MB)
bun build src/main.tsx --outdir=dist --target=bun
```

### Run Modes

```bash
# Headless print mode (no TTY needed)
bun src/main.tsx -p "your prompt here" --output-format text

# JSON output
bun src/main.tsx -p "your prompt here" --output-format json

# Interactive REPL mode (needs TTY)
bun src/main.tsx
```

> **Note**: If `ANTHROPIC_API_KEY` is set in your environment, it must be valid. To use OAuth instead, unset it:
> ```bash
> unset ANTHROPIC_API_KEY
> ```

---

## What Changed vs. the Original Snapshot

The original snapshot shipped **no `package.json`, no `tsconfig.json`, no lockfile, and no build scripts**. Over 100 internal/feature-gated modules were also missing from the source map.

### Build System (Reconstructed)

| File | Purpose |
|------|---------|
| `package.json` | 60+ npm dependencies reverse-engineered from ~1,900 source files |
| `tsconfig.json` | TypeScript config (ESNext + JSX + Bun bundler resolution) |
| `bunfig.toml` | Bun runtime configuration |
| `scripts/postinstall.sh` | Auto-creates `bun:bundle` runtime polyfill after `bun install` |
| `.gitignore` | Excludes `node_modules/`, `dist/`, lockfiles |

### Stub Modules (Created)

The original source imports many Anthropic-internal packages and feature-gated modules that were not included in the leak. Minimal stubs were created so the build completes:

| Category | Count | Examples |
|----------|-------|---------|
| Anthropic internal packages (`@ant/*`) | 4 | computer-use-mcp, computer-use-swift, claude-for-chrome-mcp |
| Native addons | 3 | color-diff-napi, audio-capture-napi, modifiers-napi |
| Cloud provider SDKs | 6 | Bedrock/Foundry/Vertex SDK, AWS STS, Azure Identity |
| OpenTelemetry exporters | 10 | OTLP gRPC/HTTP/Proto exporters |
| Other optional packages | 2 | sharp, turndown |
| Feature-gated source modules | ~90 | Tools, commands, services, components excluded from the source map |

### Source Fixes

| File | Change |
|------|--------|
| `src/main.tsx` | Runtime `MACRO` constant injection (compile-time define in production) |
| `src/main.tsx` | Fixed Commander.js `-d2e` short flag incompatibility |
| `src/bootstrap/state.ts` | Added missing `isReplBridgeActive()` export |
| `src/types/connectorText.ts` | Added `isConnectorTextBlock` function stub |
| `src/tools/WorkflowTool/constants.ts` | Added `WORKFLOW_TOOL_NAME` export |

---

## Architecture Overview

```
src/
├── main.tsx              # CLI entrypoint (Commander.js + React/Ink)
├── QueryEngine.ts        # Core LLM API engine
├── query.ts              # Agentic loop (async generator)
├── Tool.ts               # Tool type definitions
├── tools.ts              # Tool registry
├── commands.ts           # Command registry
├── context.ts            # System prompt context
│
├── tools/                # 40+ tool implementations
│   ├── AgentTool/        # Sub-agent spawning & coordination
│   ├── BashTool/         # Shell command execution
│   ├── FileReadTool/     # File reading
│   ├── FileEditTool/     # File editing
│   ├── GrepTool/         # ripgrep-based search
│   ├── MCPTool/          # MCP server tool invocation
│   ├── SkillTool/        # Skill execution
│   └── ...
│
├── services/             # External integrations
│   ├── api/              # Anthropic API client
│   ├── mcp/              # MCP server management
│   └── ...
│
├── memdir/               # Persistent memory system
├── skills/               # Skill system (bundled + user)
├── components/           # React/Ink terminal UI
├── hooks/                # React hooks
├── coordinator/          # Multi-agent orchestration
└── stubs/                # Stub packages for missing internals
```

### Key Systems

| System | Files | Description |
|--------|-------|-------------|
| **Agentic Loop** | `query.ts`, `QueryEngine.ts` | `while(true)` async generator: query -> tool calls -> results -> loop |
| **Memory** | `memdir/` | 4-type file-based memory (user/feedback/project/reference) with MEMORY.md index |
| **MCP** | `services/mcp/` | Model Context Protocol server management (stdio/http/sse/ws) |
| **Skills** | `skills/`, `tools/SkillTool/` | Reusable workflow templates (SKILL.md format) |
| **Agents** | `tools/AgentTool/` | Custom agent types via `.claude/agents/*.md` |
| **System Prompt** | `constants/prompts.ts` | Layered prompt: static -> dynamic -> memory -> agent |

### Extension Points (No Source Modification Needed)

| Mechanism | Location | Format |
|-----------|----------|--------|
| Custom Skills | `.claude/skills/name/SKILL.md` | YAML frontmatter + Markdown |
| Custom Agents | `.claude/agents/name.md` | YAML frontmatter + Markdown |
| MCP Servers | `.mcp.json` | JSON config |
| Hooks | `~/.claude/settings.json` | JSON event-action mappings |

---

## Feature Flags

The `bun:bundle` `feature()` function controls feature gating. In this build, all features default to **disabled**. To enable features, edit `node_modules/bundle/index.js` (auto-generated by `bun install`):

```javascript
const ENABLED_FEATURES = new Set([
  // Uncomment to enable:
  // 'KAIROS',              // Assistant mode
  // 'PROACTIVE',           // Proactive mode
  // 'BRIDGE_MODE',         // IDE bridge
  // 'VOICE_MODE',          // Voice input
  // 'COORDINATOR_MODE',    // Multi-agent coordinator
  // 'EXTRACT_MEMORIES',    // Background memory extraction
  // 'TEAMMEM',             // Team memory
])
```

---

## Tech Stack

| Layer | Technology |
|-------|-----------|
| Runtime | Bun |
| Language | TypeScript (strict) |
| Terminal UI | React + Ink |
| CLI | Commander.js |
| Validation | Zod v4 |
| Search | ripgrep |
| Protocols | MCP SDK, LSP |
| API | Anthropic SDK |
| Telemetry | OpenTelemetry |

---

## Scale

- **~1,900 source files**
- **512,000+ lines of TypeScript**
- **40+ tools**, **100+ commands**, **140+ UI components**
- **20MB** compiled bundle

---

## Disclaimer

- This repository is for **educational and research purposes only**.
- The original Claude Code source is the property of **Anthropic**.
- This repository is **not affiliated with, endorsed by, or maintained by Anthropic**.
- Original source exposure: 2026-03-31 via npm source map leak.

---

If this helps your research, please give it a ⭐!
