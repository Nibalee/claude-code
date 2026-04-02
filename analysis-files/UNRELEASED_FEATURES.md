# Every Unreleased Feature Hidden in Claude Code's Codebase — A Complete Deep Dive

I went through Claude Code's entire codebase and found **37 unreleased features** that aren't live yet. From a full remote agent execution framework to a virtual pet companion, here's everything Anthropic is building behind the scenes.

The codebase contains **86 build-time feature flags**, **400+ runtime A/B test flags** (via GrowthBook), and **14 active beta API headers**. Features are gated using Bun's compile-time dead code elimination (`feature('FLAG_NAME')` from `bun:bundle`), meaning disabled features are completely stripped from production builds.

---

## Table of Contents

1. [KAIROS — Remote Agent Execution Framework](#1-kairos--remote-agent-execution-framework)
2. [COORDINATOR_MODE — Multi-Agent Orchestration](#2-coordinator_mode--multi-agent-orchestration)
3. [ULTRAPLAN — Remote Planning via CCR](#3-ultraplan--remote-planning-via-ccr)
4. [PROACTIVE MODE — Autonomous Agent](#4-proactive-mode--autonomous-agent)
5. [VOICE_MODE — Push-to-Talk Voice Input](#5-voice_mode--push-to-talk-voice-input)
6. [DAEMON MODE — Background Server](#6-daemon-mode--background-server)
7. [WEB_BROWSER_TOOL — Built-in Browser Automation](#7-web_browser_tool--built-in-browser-automation)
8. [VERIFICATION_AGENT — Automated Quality Checker](#8-verification_agent--automated-quality-checker)
9. [Plugin System & Marketplace](#9-plugin-system--marketplace)
10. [SKILL_IMPROVEMENT — Self-Improving Skills](#10-skill_improvement--self-improving-skills)
11. [WORKFLOW_SCRIPTS — Workflow Automation](#11-workflow_scripts--workflow-automation)
12. [MONITOR_TOOL — Real-Time Process Streaming](#12-monitor_tool--real-time-process-streaming)
13. [AFK MODE — Away Detection & Auto Summary](#13-afk-mode--away-detection--auto-summary)
14. [CONTEXT_COLLAPSE — Granular Context Management](#14-context_collapse--granular-context-management)
15. [TOKEN_BUDGET — Continuation Control](#15-token_budget--continuation-control)
16. [REACTIVE_COMPACT — Error-Driven Compaction](#16-reactive_compact--error-driven-compaction)
17. [CACHED_MICROCOMPACT — Cache-Aware Cleanup](#17-cached_microcompact--cache-aware-cleanup)
18. [HISTORY_SNIP — Selective Context Removal](#18-history_snip--selective-context-removal)
19. [HISTORY_PICKER — Session History Search](#19-history_picker--session-history-search)
20. [SETTINGS SYNC — Cross-Device Settings](#20-settings-sync--cross-device-settings)
21. [EXTRACT_MEMORIES — Automatic Memory Extraction](#21-extract_memories--automatic-memory-extraction)
22. [SSH_REMOTE — Remote Session Management](#22-ssh_remote--remote-session-management)
23. [TERMINAL_PANEL — Built-in Terminal Sidebar](#23-terminal_panel--built-in-terminal-sidebar)
24. [QUICK_SEARCH — Fuzzy File Finder](#24-quick_search--fuzzy-file-finder)
25. [AUTO_THEME — Terminal-Aware Theming](#25-auto_theme--terminal-aware-theming)
26. [NATIVE_CLIENT_ATTESTATION — Anti-Fraud Protection](#26-native_client_attestation--anti-fraud-protection)
27. [NATIVE_CLIPBOARD_IMAGE — Fast Image Paste](#27-native_clipboard_image--fast-image-paste)
28. [POWERSHELL_AUTO_MODE — PowerShell in Auto Mode](#28-powershell_auto_mode--powershell-in-auto-mode)
29. [TREE_SITTER_BASH — Native Bash Parsing](#29-tree_sitter_bash--native-bash-parsing)
30. [STREAMLINED_OUTPUT — Distillation-Resistant Output](#30-streamlined_output--distillation-resistant-output)
31. [LODESTONE — Deep Link Protocol Handler](#31-lodestone--deep-link-protocol-handler)
32. [HOOK_PROMPTS — Programmatic Hook Input](#32-hook_prompts--programmatic-hook-input)
33. [CHICAGO_MCP — Computer Use as MCP Server](#33-chicago_mcp--computer-use-as-mcp-server)
34. [PERFETTO_TRACING — Performance Tracing](#34-perfetto_tracing--performance-tracing)
35. [REVIEW_ARTIFACT — Code Review Artifacts](#35-review_artifact--code-review-artifacts)
36. [FILE_PERSISTENCE — Modified File Upload](#36-file_persistence--modified-file-upload)
37. [BUDDY — Companion Sprite](#37-buddy--companion-sprite)

---

## 1. KAIROS — Remote Agent Execution Framework

**Status:** Feature-gated with 6 sub-flags | **Impact: Transformational**

KAIROS is the single largest unreleased system in Claude Code. It fundamentally transforms Claude Code from a local CLI tool into a **persistent, cloud-hosted autonomous agent platform**. Think of it as "Claude Code as a Service" — your agent can keep working after you close your laptop.

### How It Works

At its core, KAIROS enables Claude Code sessions to run **detached from your terminal** on Anthropic's cloud infrastructure (CCR — Claude Code Remote). Sessions connect via WebSocket to `wss://api.anthropic.com/v1/sessions/ws/{sessionId}/subscribe`, authenticated with OAuth and scoped to your organization.

The framework supports five remote task types:

- `remote-agent` — General background agents
- `ultraplan` — Remote planning sessions
- `ultrareview` — Automated code review
- `autofix-pr` — Auto-fix pull request issues
- `background-pr` — Background PR creation

Each remote task is tracked with rich state: session ID, command prompt, title, todo list, SDK message log, polling timestamps, and phase tracking. Task metadata is persisted to a session sidecar file so that `--resume` can resurrect long-running tasks even after you restart Claude Code.

The session lifecycle follows: `starting` → `running` → `detached` → `stopping` → `stopped`. When a remote session completes, a task notification is delivered to your local CLI with the results.

### Sub-Feature: KAIROS_BRIEF (Chat Mode)

Brief mode is a fundamental UI paradigm shift. Instead of Claude showing you raw tool calls, file edits, and bash commands, it restricts output to the `SendUserMessage` tool — essentially creating a **chat mode** where Claude just talks to you about what it's doing.

**How to activate:**

- `--brief` CLI flag
- `/brief` slash command (toggle mid-session)
- `--tools SendUserMessage` (implicit brief)
- `defaultView: 'chat'` in settings
- `/config` defaultView picker
- `CLAUDE_CODE_BRIEF` env var

The entitlement check cascades: KAIROS active OR env var OR GrowthBook gate `tengu_kairos_brief` (5-minute refresh TTL as kill-switch). When toggled mid-session, a `<system-reminder>` is injected explaining the mode switch, and the prompt cache is invalidated since the tool list changes.

**Impact:** Makes Claude Code accessible to non-technical users who don't want to see code diffs and terminal output. You just chat, and Claude works.

### Sub-Feature: KAIROS_CHANNELS (MCP Push Notifications)

Channels let MCP servers **push messages into your Claude Code session** in real-time. This is the foundation for integrating external services like Slack, GitHub, email, or any custom webhook into your coding workflow.

The gate sequence is deliberately strict (7 layers):

1. **Capability check**: Server declares `experimental['claude/channel']: {}`
2. **Runtime gate**: GrowthBook `tengu_harbor` (5-min TTL kill-switch)
3. **OAuth check**: Must be authenticated (not API keys)
4. **Team policy**: Enterprise/Teams must set `channelsEnabled: true` in managed settings
5. **Session opt-in**: `--channels` flag required per session
6. **Marketplace verification**: Plugin version matches installed tag
7. **Allowlist check**: Plugin in `tengu_harbor_ledger` GrowthBook array

Messages arrive wrapped in XML: `<channel source="serverName" attr="val">content</channel>`. Channels can also declare permission capabilities for structured permission request/response flows (instead of text-parsing).

**Impact:** Imagine getting a Slack message while coding, and Claude reads it, understands the context, and helps you respond — all without leaving your terminal. Or GitHub Actions failures automatically triggering Claude to investigate and fix.

### Sub-Feature: KAIROS_DREAM (Auto Memory Consolidation)

Dream mode is Claude Code's **automatic journal system**. After enough sessions pass, it reviews your recent work and distills insights into persistent memory files.

**Trigger thresholds** (from GrowthBook `tengu_onyx_plover`):

- At least 24 hours since last consolidation
- At least 5 sessions touched since last consolidation
- No other process mid-consolidation (lock-based coordination)

When triggered, a forked agent reviews recent session transcripts and updates `MEMORY.md` topic files with distilled knowledge. The task tracks two phases: `starting` → `updating` (flips on first Edit/Write), with a max of 30 turns and abort support.

In KAIROS/Assistant mode, there's also a **daily log mode**: append-only logs at `<memory>/logs/YYYY/MM/YYYY-MM-DD.md` that the nightly `/dream` skill distills into topic files.

**Impact:** Your Claude Code gets smarter over time without you manually telling it things. Project context, your preferences, architectural decisions — all automatically captured and recalled in future sessions.

### Sub-Feature: KAIROS_PUSH_NOTIFICATION (Terminal Notifications)

Push notifications alert you when background tasks complete while you're away from your terminal.

**Auto-detection by terminal:**

- **iTerm2**: Native notification API (optionally with terminal bell)
- **Kitty**: OSC 9 sequence
- **Ghostty**: Native API
- **Apple Terminal**: Checks if bell is disabled via osascript
- **Other**: Falls back to no notification

Configurable via `preferredNotifChannel` setting. Optional `taskCompleteNotifEnabled` pushes when the agent goes idle. Every notification is analytics-tracked with the configured vs. actual channel used.

**Impact:** Fire off a long task, switch to another window, and get pinged when it's done. No more polling your terminal.

### Sub-Feature: Cron/Scheduled Tasks

A full scheduling system persisted in `.claude/scheduled_tasks.json`.

**Task structure:**

```
id: 8-char UUID slice
cron: "M H DoM Mon DoW" (5-field, local time)
prompt: the text to enqueue when firing
recurring: boolean (repeat vs. one-shot)
permanent: boolean (exempt from age-out)
durable: boolean (file-backed vs. session-scoped)
```

**Distributed coordination:** A scheduler lock prevents double-firing across concurrent sessions. Lock uses PID-based liveness detection — non-owners re-probe every 5 seconds to take over if the owner crashes.

**Jitter system:** Per-task deterministic jitter (based on taskId hash) prevents thundering herd when multiple tasks share the same cron schedule. Recurring tasks get forward delay; one-shot tasks get backward lead. All tunable via GrowthBook `tengu_kairos_cron_config`.

**Missed task detection:** One-shot tasks that fire while Claude was offline are surfaced at startup with a confirmation dialog.

**Two storage modes:**

- **File-backed**: shared across sessions, persists in `.claude/scheduled_tasks.json`
- **Session-scoped**: in-memory only, dies with the process

**Impact:** Schedule Claude to run your test suite every morning at 8am, check for dependency updates weekly, or monitor a production log every 5 minutes. True automation.

### User Benefits Summary

- **Work asynchronously**: Start a task, close your laptop, come back to results
- **Integrate external services**: Slack, GitHub, email flowing into your coding session
- **Automate recurring work**: Cron-based scheduling for tests, reviews, monitoring
- **Get notified**: Push notifications when tasks complete
- **Build memory over time**: Auto-dream consolidates learnings across sessions
- **Chat mode**: Non-technical stakeholders can interact without seeing code

---

## 2. COORDINATOR_MODE — Multi-Agent Orchestration

**Status:** Feature-gated + env var | **Impact: Major Productivity Multiplier**

Coordinator mode transforms Claude Code from a single-agent system into a **multi-agent orchestration platform** where a coordinator Claude directs multiple worker Claudes in parallel.

### How It Works

When enabled (feature flag `COORDINATOR_MODE` + environment variable `CLAUDE_CODE_COORDINATOR_MODE`), Claude's entire system prompt changes. Instead of being a coding assistant, it becomes an **orchestrator** with four defined roles:

1. **Orchestrate** — Direct workers to research, implement, and verify
2. **Synthesize** — Read worker findings and understand problems
3. **Communicate** — Update the user with results
4. **Self-serve** — Answer questions directly when possible (don't over-delegate)

The coordinator gets three tools:

- `Agent()` — Spawn a new worker agent
- `SendMessage()` — Send a message to an existing worker (resume if stopped)
- `TaskStop()` — Kill a running worker

Workers receive a restricted tool pool (`ASYNC_AGENT_ALLOWED_TOOLS`): Read, Grep, Glob, Bash/PowerShell, Edit, Write, Notebook Edit, Skill, Web Search/Fetch. They explicitly cannot spawn their own sub-agents, access TaskOutput/TaskStop, or use recursive operations.

### Multi-Phase Workflow

The system prompt guides the coordinator through structured phases:

| Phase | Who Does It | Concurrency |
|-------|------------|-------------|
| **Research** | Multiple workers | Fully parallel |
| **Synthesis** | Coordinator alone | Sequential |
| **Implementation** | Workers | Serialized per file set |
| **Verification** | Workers | Can run alongside implementation on different areas |

The coordinator must craft **self-contained prompts** for each worker since workers can't see the coordinator's conversation. The system prompt explicitly calls out the anti-pattern of lazy delegation:

- Bad: *"Based on your findings, fix the bug"*
- Good: *"Fix the null pointer in src/auth/validate.ts:42. The user field on Session is undefined when sessions expire but token is cached. Add null check before user.id access — if null, return 401 with 'Session expired'. Commit and report hash."*

### Agent Communication

**Named agents** are registered in `agentNameRegistry` (a Map of name to agentId in AppState), making them addressable:

- **Running agents**: Messages queued via `queuePendingMessage()` — delivered at the worker's next check-in
- **Stopped agents**: Auto-resumed via `resumeAgentBackground()` with the queued message
- **Evicted agents**: Resumed from disk transcript if the agent was garbage collected

### Shared Knowledge

Workers share a **scratchpad directory** for cross-worker durable knowledge storage. Worker A can write research findings to the scratchpad, and Worker B can read them for implementation.

### Task Notification Protocol

When a worker finishes, a structured XML notification is delivered to the coordinator:

```xml
<task-notification>
  <task-id>agent-xyz</task-id>
  <status>completed|failed|killed</status>
  <result>agent's final text response</result>
  <usage>
    <total_tokens>N</total_tokens>
    <tool_uses>N</tool_uses>
    <duration_ms>N</duration_ms>
  </usage>
  <worktree>
    <worktreePath>/path</worktreePath>
    <worktreeBranch>branch-name</worktreeBranch>
  </worktree>
</task-notification>
```

### UI: CoordinatorAgentStatus Panel

A real-time panel shows all active agents:

```
main
researcher: Investigate auth bug     12s  2.4k  2
implementer: Fix validation logic     8s  1.1k
```

Each line shows: bullet indicator, name, description, play/pause icon, elapsed time, token count, pending message queue count. Updates every second. Selection hint shows "x to stop/clear".

### FORK_SUBAGENT (Alternative Approach)

**Mutually exclusive** with COORDINATOR_MODE. When `subagent_type` is omitted from an `Agent()` call, fork mode implicitly activates.

Key differences:

| Aspect | COORDINATOR_MODE | FORK_SUBAGENT |
|--------|------------------|---------------|
| Context | Workers start blank | Fork inherits full conversation + system prompt |
| Orchestration | Explicit via coordinator | Implicit |
| Tool pool | Restricted subset | Parent's exact tools |
| Async model | Configurable | Always async |
| Recursion | No guard needed | `FORK_BOILERPLATE_TAG` prevents infinite recursion |

Fork mode is simpler for narrow tasks where the child needs full context, while Coordinator mode is better for complex multi-file projects requiring structured decomposition.

### User Benefits

- **Parallel research**: 5 workers exploring different parts of your codebase simultaneously
- **Faster large refactors**: Split a 20-file change across multiple workers
- **Independent verification**: Spawn a fresh agent to verify another agent's work
- **Token efficiency**: Workers only load what they need instead of carrying full conversation context
- **Structured workflow**: Research, Synthesize, Implement, Verify prevents sloppy shortcuts

---

## 3. ULTRAPLAN — Remote Planning via CCR

**Status:** Feature-gated | **Impact: Better Plans for Complex Tasks**

Ultraplan offloads the planning phase to a remote Claude Code session running on the web (CCR), giving Claude a full browser-based environment to explore, think, and design before committing to local execution.

### How It Works

**Step 1: Launch**

User types `/ultraplan <prompt>` or the keyword "ultraplan" is detected in their input. Keyword detection filters aggressively to avoid false positives — it skips quoted ranges (backticks, brackets), path-like contexts (preceded/followed by `/`, `\`, `-`), question marks ("ultraplan?"), and slash command input.

**Step 2: Remote Session Creation**

Claude Code calls `teleportToRemote()` which creates a CCR session with `permissionMode: 'plan'`. Two source modes:

- **GitHub** (default): CCR backend clones from your repo's origin URL
- **Bundle** (`CCR_FORCE_BUNDLE=1`): CLI creates `git bundle --all`, uploads via Files API

The session uses the model from `getUltraplanModel()` (Opus 4.6 via first-party API).

**Step 3: 30-Minute Polling Loop**

A detached polling loop checks CCR events every 3 seconds via the Sessions API. The `ExitPlanModeScanner` class ingests SDK events and classifies them with this precedence: `approved` > `terminated` > `rejected` > `pending` > `unchanged`.

Phase transitions the user sees:

- `running` → `needs_input` (turn ends without ExitPlanMode — user needs to reply in browser)
- `needs_input` → `running` (user replied)
- `running` → `plan_ready` (ExitPlanMode tool call emitted, awaiting browser approval)
- `plan_ready` → `running` (user rejected — they can iterate more in browser)
- `plan_ready` → resolves (user approved)

**Step 4: Execution**

Two paths after plan approval:

- **Remote execution**: User approves in browser → CCR continues coding → session archived → notification sent
- **Local teleport**: User clicks "teleport back to terminal" → plan arrives as `UltraplanChoiceDialog` → user chooses to execute locally

### Related Features

- **CCR_AUTO_CONNECT** (gate: `tengu_cobalt_harbor`): Auto-connects every CLI session to CCR at startup
- **CCR_MIRROR** (gate: `tengu_ccr_mirror`): Every local session spawns an outbound-only remote session that receives forwarded events — essentially a cloud mirror of your local work
- **CCR_REMOTE_SETUP**: Enables the `/web` command for manual CCR setup

### User Benefits

- **Better plans for complex tasks**: The remote session can explore your full codebase with fresh context instead of consuming your local context window
- **Iterative refinement**: Reject a plan, give feedback, get a better one — all in the browser
- **Choose your execution environment**: Execute remotely (fire and forget) or locally (hands-on)
- **30-minute deep thinking**: The planning session gets a full 30 minutes instead of rushing

---

## 4. PROACTIVE MODE — Autonomous Agent

**Status:** Feature-gated (`PROACTIVE` or `KAIROS`) | **Impact: Hands-Free Coding**

Proactive mode enables Claude to **take initiative** — exploring, acting, and making progress without waiting for user instructions. It transforms Claude from a reactive assistant into an autonomous developer.

### How It Works

**Activation:**

- `--proactive` CLI flag
- `CLAUDE_CODE_PROACTIVE` env var
- Must be activated BEFORE `getTools()` so `SleepTool.isEnabled()` passes

When activated, a specialized system prompt is injected:

> *"You are in proactive mode. Take initiative — explore, act, and make progress without waiting for instructions. Start by briefly greeting the user. You will receive periodic `<tick>` prompts. These are check-ins. Do whatever seems most useful, or call Sleep if there's nothing to do."*

### The Tick Loop

Claude receives periodic `<tick>` prompts as check-ins. On each tick, it can:

- Continue working on the current task
- Start exploring something new
- Call `SleepTool` to wait (costs an API call but doesn't hold a shell process)

The SleepTool is cache-aware: the prompt cache expires after 5 minutes of inactivity, so sleep duration balances API cost vs. cache freshness.

### Session Behavior

- **Pauses** when user cancels input (so they get control back)
- **Resumes** on next user submit
- **Compaction aware**: After compaction, the continuation prompt says *"You are running in autonomous/proactive mode. This is NOT a first wake-up — you were already working autonomously before compaction. Continue your work loop."*
- **Onboarding skipped**: In proactive mode, example prompts are hidden since the model drives the conversation

### Integration with AGENT_TRIGGERS

Proactive mode pairs with scheduled tasks. The proactive agent can call `ScheduleCronTool` to create future tasks, and when those cron tasks fire, the prompt is enqueued as if the user submitted it — triggering the next tick cycle.

Three scheduling tools: `CronCreateTool`, `CronDeleteTool`, `CronListTool` — accepting 5-field cron expressions, with recurring or one-shot options, and durable (file-backed) or session-scoped storage.

### User Benefits

- **Background automation**: Set Claude loose on a codebase cleanup while you do other work
- **Continuous monitoring**: Proactive agent watches logs, detects issues, fixes them
- **Exploration**: Ask Claude to explore a new codebase and report back what it finds
- **Pair programming**: Claude proactively suggests improvements while you code

---

## 5. VOICE_MODE — Push-to-Talk Voice Input

**Status:** Feature-gated, fully implemented | **Impact: Hands-Free Interaction**

A complete push-to-talk voice input system that lets you speak to Claude Code instead of typing.

### Audio Capture

**Native module** (`audio-capture-napi`): Uses cpal for cross-platform audio capture:

- macOS: CoreAudio
- Linux: ALSA
- Windows: Windows audio API

**Fallback chain**: Native → `arecord` (ALSA on Linux) → SoX `rec` command

**Recording specs**: 16kHz sample rate, 16-bit signed, mono PCM format

**Silence detection**: Auto-stops recording after 2.0 seconds of silence at 3% amplitude threshold

**macOS optimization**: Lazy-loads native module on first voice keypress (not at startup) to avoid the TCC permission dialog appearing when you don't need voice.

### Speech-to-Text Pipeline

Connects to Anthropic's WebSocket endpoint: `/api/ws/speech_to_text/voice_stream`

- **STT engine**: Deepgram Nova 3 (routed via conversation_engine, gated by `tengu_cobalt_frost`)
- **Protocol**: JSON control messages (`KeepAlive`, `CloseStream`) + binary audio frames over WebSocket
- **Language support**: 19+ languages with normalization — English, Spanish, French, German, Portuguese, Japanese, and more
- **Custom keyterms**: Supports STT boosting for domain-specific vocabulary
- **Finalization**: 1.5s no-data timer for normal finalization, 5s safety timer as fallback

### Hold-to-Talk Interaction

- **Default key**: Spacebar (configurable via keybindings)
- **Recording states**: `idle` → `warmup` → `recording` → `processing`
- **Auto-repeat handling**: Detects keyboard auto-repeat via 200ms gap detection between key events. Modifier combo first-press fallback: 2000ms (covers macOS "Long" key repeat delay)
- **Text insertion**: Transcribed text inserted at cursor position without clobbering surrounding content. Handles interim transcripts for live preview (streaming updates as you speak). Auto-finalizes on segment boundaries. Strips trailing hold-key characters that leak past event propagation

### Gating

- Build-time: `feature('VOICE_MODE')`
- GrowthBook kill-switch: `tengu_amber_quartz_disabled`
- OAuth required (not available with API keys, Bedrock, Vertex, or Foundry)

### User Benefits

- **Hands-free coding**: Describe what you want while looking at code on another screen
- **Accessibility**: Users who can't type efficiently can still use Claude Code
- **Speed**: Speaking is 3-4x faster than typing for describing complex changes
- **Natural interaction**: Explain bugs, describe features, or give feedback conversationally

---

## 6. DAEMON MODE — Background Server

**Status:** Feature-gated (`DAEMON` + `BRIDGE_MODE`) | **Impact: Always-On Claude**

Daemon mode turns Claude Code into a **persistent background service** that runs continuously, accepting and processing requests without a terminal UI.

### Architecture

**Supervisor/Worker model**: A supervisor process spawns worker processes, keeping them memory-efficient for long-running operation.

- **CLI entry**: `claude daemon [subcommand]` (or `--daemon-worker=<kind>` for internal supervisor-spawned workers)
- **Direct Connect API**: `POST /sessions` → creates session → returns `session_id`, `ws_url`, `work_dir`
- **Session persistence**: Stored in `~/.claude/server-sessions.json`

### Multi-Session Support

Gated by `tengu_ccr_bridge_multi_session`:

- Spawn modes: `--spawn`, `--capacity`, `--create-session-in-dir`
- Default capacity: 32 concurrent sessions (configurable)
- Idle timeout: sessions expire after configured duration
- Capacity wake: signal triggers when a session completes so a new one can start immediately

### Token Management

- OAuth token refresh scheduler with 3h55m+ lifecycle
- Session ingress JWT tokens stored separately from OAuth tokens
- Trusted device tokens for long-lived authentication

### Bridge Mode (Remote Control)

Bridge mode is the foundation enabling CCR connectivity. `isBridgeEnabled()` checks:

- Valid claude.ai subscription (OAuth token)
- GrowthBook gate: `tengu_ccr_bridge`
- Not using Bedrock/Vertex/Foundry or env-var API keys

Permission bridge creates synthetic AssistantMessages for tool permissions between local CLI and remote CCR. The SDK message adapter handles 10+ message types: assistant, user, stream_event, result, system, tool_progress, auth_status, tool_use_summary, rate_limit_event.

### WebSocket Protocol

- URL: `wss://api.anthropic.com/v1/sessions/ws/{sessionId}/subscribe`
- Reconnection: 2s base delay, exponential backoff, max 5 attempts
- Permanent error codes (4003 unauthorized): stop immediately
- Transient errors (4001 session not found): max 3 retries
- Ping interval: 30s keep-alive

### User Benefits

- **Always-on agent**: Claude running in the background, ready to process requests
- **IDE integration**: VS Code, JetBrains, or any editor can send requests to the daemon
- **Multiple concurrent sessions**: Handle 32+ tasks simultaneously
- **Remote development**: Access your Claude from any device via the bridge

---

## 7. WEB_BROWSER_TOOL — Built-in Browser Automation

**Status:** Feature flag defined, NOT IMPLEMENTED | **Impact: Visual Web Interaction**

### What Exists

The infrastructure is wired up but the actual tool is missing:

- `tools.ts`: `require('./tools/WebBrowserTool/WebBrowserTool.js')` — **directory doesn't exist**
- `REPL.tsx`: `WebBrowserPanelModule = feature('WEB_BROWSER_TOOL')` — component reference ready
- `REPL.tsx`: `<WebBrowserPanel />` rendering slot — UI integration ready
- Runtime dependency: `'WebView' in Bun` (Bun WebView support required)

### Current Alternatives

- **Claude in Chrome** (MCP-based): Full Chrome extension MCP server with tools for navigation, form filling, screenshots, GIF recording, console logs, and network debugging. Auto-loaded when `WEB_BROWSER_TOOL` is enabled via `setupClaudeInChrome()`
- **Playwright plugin**: Install via `/plugin install playwright@claude-plugins-official` for programmatic browser automation

### What It Would Enable

- Inline browser within Claude Code's terminal UI (via Bun WebView)
- Visual debugging: See what your web app looks like without leaving the terminal
- Form testing: Claude fills out forms and checks responses
- Screenshot comparison: Before/after visual regression testing

### User Benefits (When Implemented)

- **End-to-end testing**: Claude can browse your app, click buttons, fill forms, verify output
- **Visual verification**: See rendering bugs without opening a browser manually
- **Web scraping**: Extract data from web pages as part of coding workflows
- **API testing**: Test endpoints with actual browser requests

---

## 8. VERIFICATION_AGENT — Automated Quality Checker

**Status:** Feature-gated + GrowthBook (`tengu_hive_evidence`) | **Impact: Catch Bugs Before They Ship**

A specialized built-in agent that **actually runs your code** to verify correctness — not just code review, but execution-based verification.

### How It's Triggered

When you close 3+ tasks without a verification step, the `TodoWriteTool` and `TaskUpdateTool` send a nudge recommending verification.

### Verification Philosophy

The system prompt explicitly calls out two failure patterns:

1. **Verification avoidance**: Reading code instead of running it
2. **Being seduced by the first 80%**: Declaring success too early

The agent is **required** to:

- Build the code
- Run the test suite
- Run linters/type-checkers
- Actually execute the changed functionality
- Test edge cases and error handling
- Run at least one **adversarial probe** (concurrency, boundary, idempotency, or orphan operation)

### Strategies by Change Type

| Change Type | Verification Strategy |
|-------------|----------------------|
| **Frontend** | Dev server + browser automation + console checks + subresource testing |
| **Backend/API** | Run server + hit endpoints + check responses |
| **CLI/Script** | Execute with sample input and verify output |
| **Infrastructure** | Deploy to staging + smoke test |
| **Library** | Compile + run consumer test cases |
| **Bug fixes** | Reproduce bug → apply fix → confirm fix → check regression |
| **Database migration** | Run migration + verify schema + check rollback |
| **Refactoring** | Full test suite + behavior comparison |
| **Mobile** | Build + launch emulator + UI verification |
| **Data/ML pipeline** | Run pipeline + validate output metrics |

### Restrictions

- Cannot modify project files — only writes ephemeral test scripts to `/tmp`
- Allowed tools: Read, Glob, Grep, Bash (for running things)
- Disallowed: Agent, Edit, Write (can't change your code)
- Output: Strict `VERDICT: PASS / FAIL / PARTIAL` structure

### User Benefits

- **Catch bugs before commit**: Automated testing of your changes
- **Adversarial testing**: Finds edge cases you wouldn't think to test
- **Change-type-aware**: Knows how to test a frontend change differently from a database migration
- **Non-destructive**: Can't accidentally modify your code during verification

---

## 9. Plugin System & Marketplace

**Status:** Core system exists, marketplace partially implemented | **Impact: Extensible Claude Code**

### Plugin Architecture

Plugins can provide **six types of extensions**:

- **Commands**: Custom slash commands (`/my-command`)
- **Agents**: Custom AI agent types for the Agent tool
- **Skills**: Reusable workflow prompts
- **Hooks**: Pre/post-action hooks (pre-commit, post-edit, etc.)
- **MCP Servers**: Model Context Protocol server integrations
- **Output Styles**: Custom formatting for Claude's output

### Plugin Directory Structure

```
my-plugin/
  plugin.json          # Manifest with metadata
  commands/            # Custom slash commands
  agents/              # Custom AI agents
  hooks/               # Hook configurations
  skills/              # Custom skills
```

### Installation Sources

- **Official Anthropic marketplace** (GCS-backed)
- **GitHub repositories**
- **npm packages**
- **Local file paths**

Storage: `~/.claude/plugins/`

### Marketplace UI

`BrowseMarketplace.tsx` provides a full browsing experience with pagination, plugin details, and install flow. Additional UIs include: `ManagePlugins.tsx` for managing installed plugins, `ManageMarketplaces.tsx` for marketplace sources, `DiscoverPlugins.tsx` for discovery, `PluginSettings.tsx` for configuration, `PluginTrustWarning.tsx` for security warnings, and `ValidatePlugin.tsx` for validation.

### Security

- **Trust warnings** before installation
- **Policy enforcement** via `pluginPolicy.ts`
- **Blocklist** for known-dangerous plugins
- **Marketplace verification** for channel integrations

### What's Incomplete

The code contains a TODO: *"Actually scan local plugin directories"* — the local discovery mechanism is not fully implemented.

### User Benefits

- **Customize Claude Code**: Add domain-specific commands, agents, and workflows
- **Share workflows**: Publish plugins for your team or the community
- **Integrate tools**: Connect Claude to any service via MCP plugins
- **Standardize practices**: Team-wide hooks and skills via managed plugins

---

## 10. SKILL_IMPROVEMENT — Self-Improving Skills

**Status:** Feature-gated + GrowthBook (`tengu_copper_panda`) | **Impact: Skills That Learn**

### How It Works

The skill improvement system monitors your interactions during skill execution, looking for corrections and preferences:

1. **Detection**: Every 5 turns during skill execution, an API hook analyzes your messages for correction patterns ("no, not that — instead do...", "don't...", "let's not...")
2. **Analysis**: Identifies what the user prefers vs. what the skill did
3. **Suggestion**: Creates `SkillUpdate` objects with `{section, change, reason}`
4. **UI Survey**: `SkillImprovementSurvey` component asks if you want to apply the improvement
5. **Application**: Writes changes to `.claude/skills/<name>/SKILL.md`

### User Benefits

- **Skills get better over time**: Your corrections are permanently captured
- **No manual editing**: The system detects and applies improvements automatically
- **Reason tracking**: Each improvement records why it was made
- **Team learning**: Shared skill files improve for everyone

---

## 11. WORKFLOW_SCRIPTS — Workflow Automation

**Status:** Feature-gated, implementation not present | **Impact: Multi-Step Automation**

### What the Code References

Conditional imports exist for:

- `WorkflowTool` — main tool interface
- `LocalWorkflowTask` — background task type with kill/skip/retry
- `WorkflowDetailDialog` — UI showing workflow progress
- `createWorkflowCommand()` — command factory

Functions referenced: `killWorkflowTask()`, `skipWorkflowAgent()`, `retryWorkflowAgent()`

Blocked from recursive execution — `WORKFLOW_TOOL_NAME` is in the subagent disallowed tools list.

### Intended Purpose

Multi-step workflow automation with step-by-step execution, skip/retry for individual steps, background execution with progress UI, and kill support for long-running workflows.

### User Benefits (When Implemented)

- **Complex automation**: Define multi-step workflows (lint, test, build, deploy)
- **Error recovery**: Skip or retry individual failed steps
- **Background execution**: Workflows run as background tasks
- **Reusable**: Define once, run many times

---

## 12. MONITOR_TOOL — Real-Time Process Streaming

**Status:** Feature-gated | **Impact: Better Background Process Handling**

### How It Works

The Monitor tool streams output from long-running background processes in real-time, replacing the common but wasteful pattern of `run_in_background` + sleep/poll loops.

**Key behavior:**

- Each stdout line from a monitored process triggers a notification
- MCP servers can be monitored for real-time event updates
- Task type: `MonitorMcpTask` with kill support

**Sleep blocking**: When Monitor tool is enabled, `sleep N` where N >= 2 is **rejected** as the first command in a Bash call. The system prompt instructs Claude to use Monitor instead.

**Priority**: Monitor task notifications get `'next'` priority (immediate delivery) vs. `'later'` priority for regular background tasks.

### User Benefits

- **Real-time feedback**: See output from long-running commands as it happens
- **Efficient**: No wasted API calls on sleep/poll loops
- **MCP monitoring**: Watch MCP server events in real-time
- **Immediate alerts**: Higher priority notifications than background tasks

---

## 13. AFK MODE — Away Detection & Auto Summary

**Status:** Feature-gated (`TRANSCRIPT_CLASSIFIER`), ant-only | **Impact: Seamless Resumption**

### AFK Detection

The transcript classifier detects when you're away from your terminal and switches Claude to autonomous decision-making mode with the prompt: *"The user is away. Lean heavily into autonomous action."*

Beta header: `afk-mode-2026-01-31`

### Away Summary Generation

**Trigger**: 5-minute blur delay on terminal focus loss

**Process:**

1. Takes last 30 messages for context
2. Uses small/fast model for cheap generation
3. Pulls session memory context for broader awareness
4. Generates 1-3 sentence recap: *"Start by stating the high-level task — what they are building or debugging, not implementation details. Next: the concrete next step."*

**Smart behaviors:**

- **Deduplication**: Won't generate duplicate summaries since last user turn
- **Mid-turn aware**: If timer fires mid-turn, defers until turn completes
- **Abort on return**: Cancels in-flight generation if you refocus the terminal
- **Persistence**: Creates `away_summary` system message injected into conversation

### User Benefits

- **Instant context recovery**: Come back from lunch and immediately see where you left off
- **No re-reading**: Summary tells you the task AND the next step
- **Autonomous while away**: Claude keeps working while you're gone
- **Lightweight**: Uses a small model so it's fast and cheap

---

## 14. CONTEXT_COLLAPSE — Granular Context Management

**Status:** Feature-gated | **Impact: Smarter Context Window Usage**

### How It Works

Context collapse is a more sophisticated alternative to full conversation compaction. Instead of summarizing the entire conversation at once, it **archives individual message groups** into summary placeholders.

**Mechanism:**

1. **Commit system**: Archives old messages via `ContextCollapseCommitEntry` (type: `'marble-origami-commit'`)
2. **Staged queue**: Maintains a queue of messages staged for collapsing via `ContextCollapseSnapshotEntry` (type: `'marble-origami-snapshot'`)
3. **Threshold**: Initiates at ~90% context utilization
4. **Block threshold**: Blocks new agent spawns at 95%
5. **Persistence**: Commits and snapshots stored in transcript for session restore

**Key optimization**: When context collapse is enabled, it **suppresses autocompact**. Collapse's granular message-level archiving is preferred over full conversation compaction. Autocompact becomes a fallback 413 handler only.

Includes `CtxInspectTool` for debugging context state when enabled.

### User Benefits

- **More effective context usage**: Keep important messages, archive less relevant ones
- **Gradual degradation**: Context shrinks gracefully instead of one big compaction
- **Preserves recent work**: Unlike full compaction, recent messages stay intact
- **Session resilience**: Commits persist in transcript for restore

---

## 15. TOKEN_BUDGET — Continuation Control

**Status:** Feature-gated | **Impact: Precise Task Scoping**

### How It Works

Token budgeting lets you set a token limit for multi-turn agentic work, and Claude auto-continues until the budget is consumed.

**Syntax**: `+500k`, `+2m`, or `use 500k tokens` (parsed via regex)

**BudgetTracker** tracks: continuation count, last delta tokens per turn, turn duration.

**Completion logic:**

- **Continue** if turn tokens < 90% of budget
- **Stop** at 90% OR when diminishing returns detected (delta < 500 tokens for 3+ continuations)
- **Nudge message**: *"Stopped at X% of token target (N / Budget). Keep working — do not summarize."*

**API integration**: Task budget passed as `output_config.task_budget` (beta: `task-budgets-2026-03-13`)

**UI**: Budget expressions highlighted in blue in the prompt input.

### User Benefits

- **Scoped work**: Tell Claude "use 500k tokens on this" instead of hoping it finishes
- **Auto-continuation**: No need to manually type "keep going"
- **Diminishing returns detection**: Stops automatically when progress stalls
- **Cost control**: Know exactly how many tokens a task will consume

---

## 16. REACTIVE_COMPACT — Error-Driven Compaction

**Status:** Feature-gated | **Impact: Leaner Context Management**

### How It Works

Reactive compaction is a "trap-then-handle" approach. Instead of proactively monitoring context size, it **waits for the API to return a 413 (prompt_too_long) error**, then compacts.

**Trigger conditions:**

- 413 prompt too long errors
- Media size limit errors

When enabled, proactive `shouldAutoCompact()` is suppressed. Reactive catch handles everything. Also serves as a fallback even when CONTEXT_COLLAPSE is the primary system.

### User Benefits

- **Fewer unnecessary compactions**: Only compact when actually needed
- **No wasted API calls**: Don't spend tokens checking if compaction is needed
- **Safety net**: Catches edge cases where proactive monitoring might miss

---

## 17. CACHED_MICROCOMPACT — Cache-Aware Cleanup

**Status:** Feature-gated | **Impact: Cheaper Token Usage**

### How It Works

The problem: deleting old tool results normally invalidates the prompt cache, causing expensive re-tokenization. Cached microcompact uses the `cache_edits` API to delete results **without invalidating the prefix cache**.

**Process:**

1. Registers tool results as they appear via `registerToolResult()`
2. Groups by user message position via `registerToolMessage()`
3. Calculates which old tools to delete based on configurable thresholds
4. Queues `cache_edits` block for the API layer
5. Pins edits after insertion so they're re-sent on subsequent calls

**Two paths:**

- **Time-based**: Clears tool results when there's a threshold gap since last assistant message
- **Cached**: Uses `cache_edits` API for cache-preserving deletion

**Safety**: Main thread only — prevents forked agents from polluting shared state.

### User Benefits

- **Lower costs**: Keep the prompt cache warm while still cleaning up old content
- **Faster responses**: Cache-preserved requests avoid re-tokenization latency
- **Automatic**: No user intervention needed

---

## 18. HISTORY_SNIP — Selective Context Removal

**Status:** Feature-gated | **Impact: Surgical Context Control**

### How It Works

History snipping lets you (or Claude) **remove specific message spans** from context while preserving session continuity.

- `SnipTool` for user-initiated snipping
- Boundary messages mark removed spans (so context remains coherent)
- Snipped messages filtered from `getAssistantMessages()` by default
- Absorbed silently in collapsed read groups (visible in verbose mode only)
- SDK projection via `snipProjection` module for headless sessions

### User Benefits

- **Remove noise**: Snip out irrelevant exploration that clutters context
- **Preserve important messages**: Keep only what matters
- **Surgical precision**: Remove specific message ranges instead of compacting everything
- **Session continuity**: Boundary markers keep the conversation coherent

---

## 19. HISTORY_PICKER — Session History Search

**Status:** Feature-gated | **Impact: Find Past Conversations**

### How It Works

A full-text search interface across session messages:

- `HistorySearchDialog` component provides the search UI
- `useHistorySearch` hook manages search state and history queries
- Triggered from PromptInput via history navigation keybinding

### User Benefits

- **Find past work**: Search across all your previous Claude Code sessions
- **Recall context**: Find that specific conversation where you solved a similar problem
- **Quick navigation**: Jump to relevant previous turns in long sessions

---

## 20. SETTINGS SYNC — Cross-Device Settings

**Status:** Feature-gated + OAuth | **Impact: Consistent Experience Everywhere**

### How It Works

**Upload (Interactive CLI to Cloud):**

- Triggers at startup when enabled
- Compares local settings with remote, uploads only changed entries
- Fire-and-forget in background

**Download (Cloud to Local):**

- Called at CCR/remote startup
- Applies remote entries to local files
- Clears settings cache after apply
- 3 retries with exponential backoff

**What syncs:**

| File | Description |
|------|-------------|
| `~/.claude/settings.json` | Global user settings |
| `~/.claude/CLAUDE.md` | Global user memory |
| `projects/{id}/.claude/settings.local.json` | Project-specific settings |
| `projects/{id}/CLAUDE.local.md` | Project-specific memory |

**API**: Bearer auth, 10s timeout, 500KB limit per file.

### User Benefits

- **Same Claude everywhere**: Settings and memory sync across desktop, laptop, remote sessions
- **No manual copying**: Changes propagate automatically
- **Project-aware**: Project-specific settings sync independently
- **Memory portability**: Your CLAUDE.md files travel with you

---

## 21. EXTRACT_MEMORIES — Automatic Memory Extraction

**Status:** Feature-gated | **Impact: Persistent Learning**

### How It Works

At the end of each complete query loop, a forked agent analyzes the conversation for memorable information:

1. **Pre-injection**: Memory manifest (existing memories) injected so the agent doesn't waste turns on `ls`
2. **Analysis**: Agent looks for user preferences, feedback, project context, and reference information
3. **Extraction**: Creates/updates memory files in `~/.claude/projects/<project>/memory/`
4. **Throttling**: Runs every N eligible turns (configurable via GrowthBook)
5. **Deconfliction**: Skipped if the main agent already wrote to memory files
6. **Tool restrictions**: Read/Grep/Glob/read-only Bash, and Edit/Write only within the memory directory
7. **Hard cap**: 5 turns maximum per extraction run

**Four memory types:**

- `user` — Role, preferences, knowledge
- `feedback` — Corrections and guidance
- `project` — Goals, deadlines, work context
- `reference` — Pointers to external systems

### User Benefits

- **Claude learns from every session**: Preferences and context captured automatically
- **No manual memory management**: The extraction agent handles it
- **Deduplication**: Won't create duplicate memories
- **Safe**: Can only write to the memory directory, nothing else

---

## 22. SSH_REMOTE — Remote Session Management

**Status:** Built for CCR | **Impact: Cloud-Based Development**

### How It Works

Full WebSocket + HTTP POST infrastructure for managing remote Claude Code sessions:

**RemoteSessionManager:**

- Manages WebSocket connection to CCR
- Routes SDK messages (model outputs) to `onMessage` callback
- Routes control requests (permissions) to `onPermissionRequest` callback
- Handles cancel requests and interrupt signals

**SessionsWebSocket:**

- URL: `wss://api.anthropic.com/v1/sessions/ws/{sessionId}/subscribe`
- Reconnection: 2s base, exponential backoff, max 5 attempts
- Permanent error detection (4003 unauthorized stops immediately)
- Transient retries (4001 session not found, max 3)
- 30s ping keep-alive

**Permission Bridge:**

- Creates synthetic AssistantMessages for tool permissions
- Creates Tool stubs for tools not in local CLI (MCP tools on CCR)
- Bridges request/response between local and remote

**SDK Message Adapter:**

Converts 10+ message types: assistant, user, stream_event, result, system, tool_progress, auth_status, tool_use_summary, rate_limit_event. Gracefully ignores unknown types.

### User Benefits

- **Remote development**: Run Claude on powerful cloud machines
- **Permission control**: Still approve tool use from your local terminal
- **Reliable connection**: Auto-reconnect with exponential backoff
- **Full fidelity**: All message types preserved across the bridge

---

## 23. TERMINAL_PANEL — Built-in Terminal Sidebar

**Status:** Feature-gated | **Impact: No More Tab Switching**

### How It Works

A built-in terminal you can toggle with **Alt+J** (Meta+J), without leaving Claude Code.

**tmux-powered:**

- Uses tmux for shell persistence
- Isolated per instance: unique socket `claude-panel-{session-uuid-first-8-chars}`
- Session name: `panel`
- Alt+J inside terminal detaches back to Claude Code
- Status bar: "Alt+J to return to Claude"

**Fallback**: Direct shell spawn if tmux is unavailable

**UI integration**: Uses Ink alternate screen to suspend Claude Code UI while terminal is shown. Lazily instantiated on first toggle.

**Cleanup**: `tmux kill-server` spawned detached on Claude Code exit, with `.unref()` so it doesn't block process exit.

### User Benefits

- **No tab switching**: Quick terminal access without leaving Claude Code
- **Persistent shell**: Shell session survives across toggles
- **Isolated**: Each Claude Code instance gets its own terminal
- **Quick toggle**: Alt+J to switch instantly

---

## 24. QUICK_SEARCH — Fuzzy File Finder

**Status:** Feature-gated | **Impact: IDE-Like File Navigation**

### How It Works

Two features:

**Quick Open (Ctrl+Shift+P / Cmd+Shift+P):**

- Fuzzy file finder using `FileIndex` with `generateFileSuggestions()`
- Shows 8 visible results (scaled to terminal size)
- Preview window: 20 lines with syntax highlighting via highlight.js
- Responsive layout adapts to terminal width
- Can insert result as `@file` mention or plain text path

**Global Search (Ctrl+Shift+F / Cmd+Shift+F):**

- Full-text search across all files

### User Benefits

- **IDE-like experience**: Ctrl+P file finder in the terminal
- **Syntax-highlighted preview**: See file contents before selecting
- **Fast fuzzy matching**: Find files by partial name
- **Quick mentions**: Insert `@file` references without typing full paths

---

## 25. AUTO_THEME — Terminal-Aware Theming

**Status:** Feature-gated | **Impact: Visual Comfort**

### How It Works

Adds "Auto (match terminal)" to the theme picker. When active, a dynamic watcher (lazy-loaded via `systemThemeWatcher.js`) monitors system/terminal theme changes and adjusts Claude Code's theme in real-time.

### User Benefits

- **Auto dark/light mode**: Matches your terminal's theme automatically
- **No manual switching**: Changes in real-time when your system theme changes
- **Consistent look**: Claude Code always matches its surroundings

---

## 26. NATIVE_CLIENT_ATTESTATION — Anti-Fraud Protection

**Status:** Feature-gated | **Impact: Security**

### How It Works

Adds a `cch=00000` placeholder to the `x-anthropic-billing-header`. Before HTTP requests are sent, Bun's native HTTP stack (written in Zig: `bun-anthropic/src/http/Attestation.zig`) finds this placeholder and overwrites the zeros with a **computed hash**.

The server verifies this token to confirm the request came from a genuine Claude Code client, not a modified or spoofed build.

**Design detail**: Same-length replacement (5 zeros to 5-char hash) avoids Content-Length changes and buffer reallocation.

### User Benefits

- **Protects your account**: Prevents unauthorized clients from using your credentials
- **No user action needed**: Transparent, automatic
- **Performance-optimized**: In-place hash replacement avoids allocation overhead

---

## 27. NATIVE_CLIPBOARD_IMAGE — Fast Image Paste

**Status:** Feature-gated + GrowthBook (`tengu_collage_kaleidoscope`) | **Impact: Instant Image Sharing**

### How It Works

Native macOS NSPasteboard reader for **instant clipboard image capture**:

- **5ms cold start, <1ms warm** vs. ~1.5 seconds for the osascript fallback (300x faster)
- Reads PNG bytes directly from the pasteboard
- Downsamples via CoreGraphics if over dimension cap
- Falls through to osascript when native module unavailable
- Size cap: 3.75MB raw / 5MB base64 API limit
- Returns metadata: originalWidth, originalHeight, displayWidth, displayHeight

Module: `image-processor-napi` with `getNativeModule()?.readClipboardImage` and `getNativeModule()?.hasClipboardImage`

### User Benefits

- **300x faster image paste**: 5ms vs 1500ms — feels instant
- **Auto-downsampling**: Large images automatically resized for the API
- **Seamless fallback**: Works everywhere, just faster on macOS with native module
- **Screenshot sharing**: Paste a screenshot and Claude sees it immediately

---

## 28. POWERSHELL_AUTO_MODE — PowerShell in Auto Mode

**Status:** Feature-gated (ant-only) | **Impact: Windows Auto Mode**

### How It Works

By default, PowerShell is **excluded from the YOLO classifier** in auto mode — every PowerShell command requires explicit user permission. This feature enables PowerShell to flow through the classifier like Bash.

**Dangerous pattern detection** when enabled:

- `iex (iwr ...)` — download-and-execute
- `Remove-Item -Recurse -Force` / `rm -r -fo` — irreversible destruction
- `$PROFILE` modification — unauthorized persistence
- `Register-ScheduledTask`, `New-Service` — service persistence
- Registry Run keys, WMI event subscriptions — system persistence

When enabled, the classifier prompt gets `POWERSHELL_DENY_GUIDANCE` appended with these patterns.

### User Benefits

- **Full auto mode on Windows**: PowerShell commands can run without manual approval
- **Safety preserved**: Dangerous patterns still blocked
- **Feature parity**: Windows users get the same auto mode experience as Unix users

---

## 29. TREE_SITTER_BASH — Native Bash Parsing

**Status:** Feature-gated (ant-only) | **Impact: Better Security Analysis**

### How It Works

Replaces regex-based bash command parsing with a proper **tree-sitter AST parser** for security analysis.

- **TREE_SITTER_BASH**: Enables the native parser for production use
- **TREE_SITTER_BASH_SHADOW**: Shadow mode where tree-sitter runs **in parallel** with the legacy parser without affecting decisions. Logs divergences for validation

**Specifications:**

- Parse timeout: 50ms
- Max nodes: 50,000
- Returns `PARSE_ABORTED` sentinel (distinct from null) for adversarial inputs
- GrowthBook kill-switch: `tengu_birch_trellis`

**What it extracts from the AST:**

- Quote context (single/double/ansi-c/heredoc)
- Compound structure (operators, pipelines, subshells, command groups)
- Dangerous patterns (command substitution, process substitution, parameter expansion, heredoc, comments)
- Subcommands and their redirects

### User Benefits

- **More accurate security decisions**: AST parsing catches tricks that regex misses
- **Fewer false positives**: Understands quoting and nesting correctly
- **Fewer false negatives**: Catches obfuscated dangerous commands
- **Safe rollout**: Shadow mode validates before switching

---

## 30. STREAMLINED_OUTPUT — Distillation-Resistant Output

**Status:** Feature-gated + env var | **Impact: Anti-Distillation**

### How It Works

Transforms SDK `stream-json` output to resist model distillation:

| Content Type | Transformation |
|-------------|----------------|
| Text messages | Kept intact |
| Tool calls | Summarized as cumulative counts |
| Thinking content | Omitted entirely |
| Init messages | Tool list and model info stripped |

**Tool categories tracked**: searches (grep, glob, web search, LSP), reads (file read, list MCP), writes (file write, edit, notebook), commands (bash/powershell/tmux, task stop), other.

**Example output**: *"Searched 3 patterns, read 2 files, ran 5 commands"*

State machine: counts accumulate tool uses, emit summary when tool-only message seen, reset on text message.

Activation: `CLAUDE_CODE_STREAMLINED_OUTPUT=true` env var.

### User Benefits

- **Protects Anthropic's models**: Makes it harder to distill Claude's behavior from output
- **Cleaner output**: Summary format is more readable than raw tool calls
- **Opt-in**: Only activates when explicitly enabled

---

## 31. LODESTONE — Deep Link Protocol Handler

**Status:** Feature-gated | **Impact: OS-Level Integration**

### How It Works

Registers Claude Code as a handler for `claude://` URIs at the OS level.

**Flow:**

1. At startup, registers protocol handler in background via `ensureDeepLinkProtocolRegistered()`
2. When a `claude://` URI is opened, handles it before full app initialization
3. Parses URI for prefill content and repository information
4. Opens appropriate terminal and launches Claude Code with the deep link content
5. Shows deep link banner when `options.deepLinkOrigin` provided
6. Tracks telemetry: `tengu_deep_link_opened` with `has_prefill` and `has_repo` flags

**Settings**: `disableDeepLinkRegistration` option to opt out.

**Stored preference**: `updateDeepLinkTerminalPreference()` remembers which terminal you use.

### User Benefits

- **Click-to-code**: Links on websites or in docs can open Claude Code with context pre-filled
- **Repository shortcuts**: Deep links can open Claude in the right project
- **Cross-app integration**: Other tools can invoke Claude Code via URI
- **Onboarding**: Share deep links that set up Claude with specific instructions

---

## 32. HOOK_PROMPTS — Programmatic Hook Input

**Status:** Feature-gated | **Impact: Interactive Hooks**

### How It Works

Exposes the `requestPrompt` function to the hook system, allowing hooks to programmatically request user input.

In REPL: `requestPrompt: feature('HOOK_PROMPTS') ? requestPrompt : undefined` — passed as part of the hook context to the query engine.

This means hooks can pause execution, ask the user a question, and use the response to make decisions.

### User Benefits

- **Interactive hooks**: Pre-commit hooks that ask "Are you sure?" with context
- **Conditional logic**: Hooks that branch based on user input
- **Confirmation flows**: Require explicit approval for specific operations
- **Custom workflows**: Build interactive automation pipelines

---

## 33. CHICAGO_MCP — Computer Use as MCP Server

**Status:** Feature-gated | **Impact: Computer Control**

### How It Works

Exposes Computer Use capabilities as an MCP server subprocess, enabling other tools to use Claude's computer control abilities.

**CLI flag**: `--computer-use-mcp`

**When enabled:**

1. Creates in-process MCP server via `createComputerUseMcpServerForCli()`
2. Sets up stdio transport for MCP communication
3. Starts app enumeration via Spotlight (macOS, 1s timeout)
4. Builds computer use tools with coordinate mode
5. Replaces ListTools handler to include installed app names in tool descriptions

**Architecture**: The MCP server only answers ListTools requests (to provide tool list with app hints). Actual tool dispatch goes through `wrapper.tsx`.

**Platform adapters**: macOS, Windows, Linux-specific implementations for keyboard/mouse input.

### User Benefits

- **GUI automation**: Control desktop applications programmatically
- **App-aware**: Knows what apps are installed and can interact with them
- **MCP protocol**: Standard interface that any MCP client can use
- **Cross-platform**: Adapters for macOS, Windows, and Linux

---

## 34. PERFETTO_TRACING — Performance Tracing

**Status:** Ant-only | **Impact: Deep Performance Insight**

### How It Works

Generates traces in Chrome Trace Event format, viewable in `ui.perfetto.dev` or `chrome://tracing`.

**Activation**: `CLAUDE_CODE_PERFETTO_TRACE=1` or `CLAUDE_CODE_PERFETTO_TRACE=<path>`

**Trace location**: `~/.claude/traces/trace-<session-id>.json`

**What gets traced:**

- Agent hierarchy (parent-child relationships in swarms)
- API requests: TTFT (time to first token), TTLT (time to last token), prompt length, cache stats, message ID, speculative flag
- Tool executions: name, duration, token usage
- User input waiting time

**Architecture:**

- Events stored in memory with phase markers (`B`/`E`/`X` for begin/end/complete)
- Max 100,000 events with LRU eviction (oldest 50% dropped)
- Pending spans tracked with 30-minute TTL, 1-minute cleanup interval
- Process/thread IDs mapped from agent names
- Optional periodic writes via `CLAUDE_CODE_PERFETTO_WRITE_INTERVAL_S`

### User Benefits

- **Diagnose slow sessions**: See exactly where time is spent
- **API performance**: Track TTFT/TTLT across requests
- **Agent hierarchy visualization**: See multi-agent orchestration in timeline view
- **Cache analysis**: Understand cache hit rates and their impact

---

## 35. REVIEW_ARTIFACT — Code Review Artifacts

**Status:** Feature-gated, mostly unimplemented | **Impact: Structured Code Review**

### What Exists

- Registers "Hunter" bundled skill when enabled
- Conditional tool import: `ReviewArtifactTool` — **directory doesn't exist**
- Permission UI: `ReviewArtifactPermissionRequest` — *"Claude needs your approval for a review artifact"*

### Intended Purpose

A structured code review system that generates review artifacts (findings, suggestions, quality assessments) requiring explicit user approval before creation.

### User Benefits (When Implemented)

- **Structured reviews**: Formal review artifacts instead of inline comments
- **Approval flow**: Review and approve findings before they're finalized
- **Audit trail**: Persistent record of review decisions
- **Skill integration**: "Hunter" skill for targeted code review

---

## 36. FILE_PERSISTENCE — Modified File Upload

**Status:** Feature-gated | **Impact: Cloud File Sync**

### How It Works

Uploads files modified during a Claude Code session to the Files API, enabling cloud-based file versioning.

**Process:**

1. `runFilePersistence()` executes at turn end with turn start timestamp
2. `findModifiedFiles()` scans for files with `mtimeMs > turnStartTime`
3. Validates paths stay within outputs directory
4. Uploads in parallel with configurable concurrency
5. Tracks successful vs. failed uploads

**Requirements:**

- `CLAUDE_CODE_REMOTE_SESSION_ID` environment variable
- OAuth session access token
- BYOC (Bring Your Own Container) mode

### User Benefits

- **Cloud versioning**: Modified files automatically uploaded for history
- **Remote session sync**: Files from CCR sessions available via API
- **Parallel uploads**: Fast, concurrent upload of multiple files
- **Audit trail**: Know exactly what files were modified in each session

---

## 37. BUDDY — Companion Sprite

**Status:** Feature-gated, partially implemented | **Impact: Delight & Personality**

BUDDY is an ASCII companion sprite that lives beside your input prompt in Claude Code. It's essentially a virtual pet / tamagotchi-style companion, unique to each user.

### How It Works

Each user gets a **deterministic companion** generated from a hash of their user ID (seeded PRNG using `userId + 'friend-2026-401'`). The companion's "bones" (species, rarity, eyes, hat, stats) are never stored — they're regenerated from the hash every time, so you can't fake a higher rarity by editing config.

**18 possible species:** duck, goose, blob, cat, dragon, octopus, owl, penguin, turtle, snail, ghost, axolotl, capybara, cactus, robot, rabbit, mushroom, chonk

**Rarity tiers:**

| Rarity | Chance | Stars |
|--------|--------|-------|
| Common | 60% | 1 star |
| Uncommon | 25% | 2 stars |
| Rare | 10% | 3 stars |
| Epic | 4% | 4 stars |
| Legendary | 1% | 5 stars |

**Customization traits:**

- **6 eye styles**: dot, star, x, bullseye, at, degree
- **8 hat options**: none, crown, tophat, propeller, halo, wizard, beanie, tinyduck
- **Shiny variant**: 1% chance
- **5 stats** (1-100): DEBUGGING, PATIENCE, CHAOS, WISDOM, SNARK

### Visual Rendering

- Each species has **3 ASCII art frames** (5 lines tall, 12 chars wide) that animate on a 500ms tick
- **Idle animation** cycles through rest, fidgets, and blinks in a 15-step sequence
- **Speech bubbles** appear next to the sprite, bordered with rarity-colored frames, lasting 10 seconds with a fade-out
- **Petting** triggers floating heart animations for 2.5 seconds
- On **narrow terminals** (< 100 cols), collapses to a single-line emoji face like `(dot>` (duck) or `=dot w dot=` (cat)

### System Prompt Integration

When active, Claude gets a system reminder explaining the companion's presence. Claude is instructed to stay out of the way when the companion speaks.

### Planned Launch Window

The code reveals a specific rollout schedule:

- **Teaser window**: April 1-7, 2026 (shows a rainbow `/buddy` notification)
- **Live**: April 2026 onwards
- Ant (internal) users can see the teaser anytime

### What's Still Missing

1. **`observer.ts`** — The file that would generate companion reactions during conversations is referenced but doesn't exist
2. **`/buddy` command** — Referenced in commands.ts but the handler directory doesn't exist yet
3. **Soul generation** — The system for Claude to generate a unique name and personality on first hatch is not yet implemented

### User Benefits

- **Personalization**: Every user gets a unique companion based on their user ID
- **Collectibility**: Rarity system (1% legendary!) creates excitement
- **Emotional connection**: Your companion reacts to your coding, can be petted with floating hearts
- **Fun**: ASCII art animations bring personality to the terminal
- **Non-intrusive**: Mutable, collapses on narrow terminals, stays out of the way

---

## Summary

The Claude Code codebase reveals an ambitious roadmap:

- **KAIROS** turns Claude Code into a persistent autonomous platform with cron jobs, channels, and push notifications
- **COORDINATOR_MODE** enables true multi-agent orchestration with named workers, shared scratchpads, and structured workflows
- **ULTRAPLAN** offloads complex planning to the cloud with iterative refinement
- **VOICE_MODE** is fully built with native audio capture and Deepgram STT
- **DAEMON MODE** makes Claude an always-on background service
- **BUDDY** adds personality with a unique companion sprite for every user

The codebase contains **86 build-time feature flags**, **400+ runtime A/B test flags**, and **14 active beta API headers** — revealing one of the most extensively feature-gated codebases in the developer tools space. Every feature can be independently rolled out, tested, and killed without touching a line of code.
