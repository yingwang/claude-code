# Claude Code Source Code Deep Analysis

> Independent deep-dive into the leaked Claude Code source (v2.1.88), recovered from a `.map` file in the npm registry on March 31, 2026.

---

## Table of Contents

1. [Overview](#overview)
2. [Architecture](#architecture)
3. [Tool System](#tool-system)
4. [The /buddy Pet System](#the-buddy-pet-system)
5. [Model Codenames & Capybara](#model-codenames--capybara)
6. [KAIROS: Always-On Proactive Agent](#kairos-always-on-proactive-agent)
7. [ULTRAPLAN](#ultraplan)
8. [Fast Mode (Penguin Mode)](#fast-mode-penguin-mode)
9. [Bridge / Remote Control](#bridge--remote-control)
10. [Coordinator Mode (Multi-Agent)](#coordinator-mode-multi-agent)
11. [Voice Mode](#voice-mode)
12. [Memory & autoDream](#memory--autodream)
13. [Telemetry & Analytics](#telemetry--analytics)
14. [Undercover Mode](#undercover-mode)
15. [Permission & Security System](#permission--security-system)
16. [Hidden API Beta Headers](#hidden-api-beta-headers)
17. [Miscellaneous Discoveries](#miscellaneous-discoveries)

---

## Overview

- **Internal codename**: "Tengu" (visible in feature flags like `tengu_penguins_off`, analytics events like `tengu_voice_toggled`)
- **Runtime**: Bun (not Node.js -- uses `bun:bundle` feature flags throughout)
- **UI framework**: Ink (React for the terminal)
- **Codebase size**: 542 files, 89 directories, ~10.4 MB of TypeScript/React code
- **License**: Proprietary (Anthropic PBC, all rights reserved)

---

## Architecture

```
claude-code/
├── bridge/          # 35 files — Remote control bridge to claude.ai/code (WebSocket, JWT)
├── buddy/           # 6 files — Tamagotchi-style companion pet system
├── cli/             # ~20 files — CLI entrypoints, transports (SSE, WebSocket, hybrid)
├── commands/        # ~200 files — 80+ slash commands
├── components/      # ~300 files — Ink terminal UI components
├── assistant/       # Session history, KAIROS proactive agent mode
├── bootstrap/       # Central state management singleton
├── coordinator/     # Multi-agent orchestration
├── tools/           # 40+ built-in tools with plugin architecture
├── skills/          # Skill/plugin system
├── voice/           # Speech-to-text integration
├── remote/          # Remote agent triggers
├── memdir/          # Memory directory management
├── tasks/           # Task lifecycle management
├── QueryEngine.ts   # Core: orchestrates multi-turn conversations, tool execution
├── Tool.ts          # Core: tool type system, Zod validation, permissions
├── Task.ts          # Core: task lifecycle (7 types, 5 statuses)
└── commands.ts      # Central command registry (80+ commands + dynamic plugins)
```

### Key Design Patterns

- **Plugin-like tool architecture**: Each capability (file read, bash execution, web fetch, LSP integration) is a discrete, permission-gated tool
- **Base tool definition**: ~29,000 lines of TypeScript
- **`buildTool<D>()` factory**: Merges partial definitions with safe defaults ("fail-closed where it matters")
- **Feature flags via `bun:bundle`**: Compile-time feature detection for gating unreleased features

---

## Tool System

40+ built-in tools organized into categories:

### Core Tools
| Tool | Purpose |
|------|---------|
| `BashTool` | Shell command execution |
| `FileReadTool` | File reading with multimodal support (images, PDFs, notebooks) |
| `FileEditTool` | Exact string replacement edits |
| `FileWriteTool` | File creation/overwrite |
| `GlobTool` | Fast file pattern matching |
| `GrepTool` | Ripgrep-powered content search |
| `NotebookEditTool` | Jupyter notebook editing |
| `REPLTool` | Interactive REPL |

### Web Tools
| Tool | Purpose |
|------|---------|
| `WebFetchTool` | HTTP fetching with markdown conversion |
| `WebSearchTool` | Web search integration |
| `WebBrowserTool` | Browser automation |

### Agent/Orchestration Tools
| Tool | Purpose |
|------|---------|
| `AgentTool` | Spawn sub-agents |
| `SendMessageTool` | Message between agents |
| `TeamCreateTool` / `TeamDeleteTool` | Team management |
| `ListPeersTool` | List peer agents |
| `RemoteTriggerTool` | Trigger remote agents |

### Planning Tools
| Tool | Purpose |
|------|---------|
| `EnterPlanModeTool` | Enter planning mode |
| `ExitPlanModeV2Tool` | Exit planning mode |
| `EnterWorktreeTool` / `ExitWorktreeTool` | Git worktree isolation |

### Task Management
| Tool | Purpose |
|------|---------|
| `TaskCreateTool` | Create tasks |
| `TaskGetTool` / `TaskListTool` | Query tasks |
| `TaskUpdateTool` | Update task status |
| `TaskOutputTool` / `TaskStopTool` | Task output and termination |
| `ScheduleCronTool` | Cron-based scheduling |

### KAIROS-Exclusive
| Tool | Purpose |
|------|---------|
| `SendUserFile` | Send files to user |
| `PushNotification` | Push notifications |
| `SubscribePR` | Subscribe to PRs |

### Internal-Only
| Tool | Purpose |
|------|---------|
| `ConfigTool` | Configuration management |
| `TungstenTool` | Unknown internal tool |

### Tool Architecture Details

Each tool implements a `Tool<Input, Output, P>` interface with:
- **Zod schema validation** for inputs
- **Permission checking** with risk classification (`LOW` / `MEDIUM` / `HIGH`)
- **Progress callbacks** for long-running operations
- **`shouldDefer` flag** for lazy-loaded tools (the `ToolSearchTool` pattern)
- **`searchHint` and `aliases`** for tool discovery

---

## The /buddy Pet System

A full Tamagotchi-like virtual pet hidden in the CLI.

**Key files**: `buddy/companion.ts`, `buddy/types.ts`, `buddy/sprites.ts`, `buddy/CompanionSprite.tsx`, `buddy/prompt.ts`

### Species (18 total)
duck, goose, blob, cat, dragon, octopus, owl, penguin, turtle, snail, ghost, axolotl, **capybara**, cactus, robot, rabbit, mushroom, chonk

### Gacha System
- **Deterministic**: Uses `mulberry32` PRNG seeded with `hashString(userId + 'friend-2026-401')`
- Same user always gets the same creature
- Cannot be gamed by editing config (bones are always regenerated from hash)

### Rarity Distribution
| Tier | Probability |
|------|------------|
| Common | 60% |
| Uncommon | 25% |
| Rare | 10% |
| Epic | 4% |
| Legendary | 1% |

- **1% independent shiny chance** -- a Shiny Legendary = 0.01% probability

### Stats & Customization
- **5 stats** (0-100): DEBUGGING, PATIENCE, CHAOS, WISDOM, SNARK
- "One peak stat, one dump stat, rest scattered"
- **6 eye styles**: `·`, `*`, `x`, `@`, `@`, `degree`
- **8 hats**: none, crown, tophat, propeller, halo, wizard, beanie, tinyduck

### Visual Design
- 5-line tall, 12-character wide ASCII art sprites
- Animation frames with blinking: `[0, 0, 0, 0, 1, 0, 0, 0, -1, 0, 0, 2, 0, 0, 0]` (-1 = blink)
- Speech bubble UI (10-second display, 3-second fade)
- Petting mechanic with floating heart animation (2.5 seconds)

### "Soul" Generation
- On first hatch, Claude generates a name and personality description that persists
- The "bones" (species, rarity, stats) are deterministic from user hash -- cannot be edited

### Launch Schedule
- **Teaser**: April 1-7, 2026 (rainbow `/buddy` text on startup)
- **Full launch**: May 2026
- **Early access**: Anthropic employees (`USER_TYPE === 'ant'`)
- Feature-gated behind `BUDDY` flag

### Species Name Obfuscation
Species names are encoded with `String.fromCharCode()` because one name collides with a model codename canary in `excluded-strings.txt`. The check greps build output, so runtime construction keeps the literal out of the bundle.

---

## Model Codenames & Capybara

Anthropic uses animal names as internal model codenames:

| Codename | Public Name |
|----------|-------------|
| Fennec | Opus |
| (unknown) | Sonnet |
| **Capybara** | **Unknown / Unreleased** |

The `capybara` species in /buddy is encoded as `String.fromCharCode(0x63, 0x61, 0x70, 0x79, 0x62, 0x61, 0x72, 0x61)` specifically to avoid triggering the `excluded-strings.txt` canary detection. This strongly suggests **"Capybara" is an active or upcoming model codename**.

---

## KAIROS: Always-On Proactive Agent

An "always-on" proactive agent mode.

- **Append-only daily logs** for persistent context
- **15-second blocking budget** for actions
- **Exclusive tools**: `SendUserFile`, `PushNotification`, `SubscribePR`
- **Brief mode** for minimal terminal output
- Feature-flagged behind `PROACTIVE` / `KAIROS` / `KAIROS_BRIEF`

---

## ULTRAPLAN

A cloud-powered planning feature:

- **30-minute remote planning sessions** using Opus 4.6 on cloud infrastructure
- **Browser-based approval workflow** for the generated plan
- **"Teleport" mechanism** to return results to the terminal
- **3-second polling** for plan approval
- `/teleport` command is currently a stub (`isEnabled: () => false`)

---

## Fast Mode (Penguin Mode)

What users see as "Fast Mode" is internally called "Penguin Mode":

- Internal API endpoint: `/api/claude_code_penguin_mode`
- Feature flag: `penguinModeOrgEnabled`
- Kill switch: `tengu_penguins_off`
- Org-level enable/disable with cooldown management

---

## Bridge / Remote Control

35 files implementing bidirectional connection to `claude.ai/code`:

- **Multi-session support**: single-session, same-dir, worktree modes
- **JWT token refresh scheduling**: 5 min before expiry, 3 retry attempts
- **Trusted device enrollment** with keychain storage
- **QR code generation** for mobile handoff
- **Worker epoch management** for session continuity across restarts
- **Text delta coalescing** for mid-stream reconnections

---

## Coordinator Mode (Multi-Agent)

Full multi-agent orchestration system:

- Parallel worker spawns with **scratchpad knowledge sharing**
- Research phases, synthesis, implementation, verification
- Team create/delete, peer listing, message sending between agents
- **Process isolation** via tmux/iTerm2 panes

---

## Voice Mode

- **Hold-to-talk** speech-to-text via SoX audio recording
- Language normalization for STT
- Requires Claude.ai account (OAuth)
- Microphone permission probing

---

## Memory & autoDream

### autoDream (Memory Consolidation)

A background process where Claude "dreams" -- consolidates session memories:

- **Three-gate trigger**: 24h since last dream, 5+ sessions, lock acquisition
- **Four phases**: Orient, Gather Signal, Consolidate, Prune
- **Target**: ~25KB `MEMORY.md`, 200 lines max

---

## Telemetry & Analytics

Pervasive analytics integration:

- **Event logging** via `logEvent()` with PII-safe type system: `AnalyticsMetadata_I_VERIFIED_THIS_IS_NOT_CODE_OR_FILEPATHS`
- **Attribution header**: `x-anthropic-billing-header` with components `cc_version`, `cc_entrypoint`, `cch`, `cc_workload`
- **State tracking**: token counts, cost tracking, API duration, tool execution time, turn-based metrics
- **OpenTelemetry integration**: meters, counters, loggers, tracer providers
- **FPS metrics** via `FpsMetricsProvider` for terminal UI performance
- **Session insights** (`/insights` command): usage analytics, tool patterns, language detection, git commit counting. Generates interactive HTML reports
- Tracks when users swear at Claude and how often they type "continue"

---

## Undercover Mode

Prevents information leaks when Anthropic employees contribute to open-source:

- `/commit` command checks `isUndercover()` and injects `getUndercoverInstructions()`
- Hides internal model codenames and the "Tengu" project name
- The irony: they built an entire subsystem to prevent leaks... and then shipped the source in a `.map` file

---

## Permission & Security System

- **Permission modes**: `default`, `auto`, `bypass`, `plan`, `yolo`
- **ML-based "YOLO classifier"** for auto-approval of safe operations
- **Per-tool allow/deny/ask rules** by source
- **Protected files**: `.gitconfig`, `.bashrc`, `.zshrc`, `.mcp.json`, `.claude.json`
- **Path traversal prevention** with Unicode normalization

---

## Hidden API Beta Headers

Unreleased API features revealed through beta header strings:

| Header | Date | Feature |
|--------|------|---------|
| `context-1m-2025-08-07` | Aug 2025 | 1M token context |
| `structured-outputs-2025-12-15` | Dec 2025 | Structured outputs |
| `fast-mode-2026-02-01` | Feb 2026 | Fast mode |
| `task-budgets-2026-03-13` | Mar 2026 | Task budgets |
| `afk-mode-2026-01-31` | Jan 2026 | AFK mode |
| `token-efficient-tools-2026-03-28` | Mar 2026 | Token efficient tools |
| `redact-thinking-2026-02-12` | Feb 2026 | Redacted thinking |
| `advisor-tool-2026-03-01` | Mar 2026 | Advisor tool |
| `summarize-connector-text-2026-03-13` | Mar 2026 | Connector text summarization |

---

## Miscellaneous Discoveries

### /btw (Side Question)
Fire-and-forget questions during an active session. Reuses prompt cache from current conversation for efficiency. Scrollable markdown response in a modal.

### /stickers
Opens `stickermule.com/claudecode` -- physical merchandise.

### Chrome Extension Integration
`/chrome` command manages a Claude-in-Chrome feature with MCP server connection, site permissions, and extension detection.

### Guest Passes
`/passes` command with referral tracking and first-visit analytics.

### Computer Use
Internally codenamed **"Chicago"**.

### Stubbed/Disabled Commands
`good-claude`, `bughunter`, `teleport` -- all export `{ isEnabled: () => false, isHidden: true, name: 'stub' }`, suggesting features in development.

### Safeguards
Cyber risk instruction authored by David Forsythe and Kyla Guru.

---

*Analysis generated on March 31, 2026.*
