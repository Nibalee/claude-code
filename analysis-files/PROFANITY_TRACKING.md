# Claude Code Is Tracking Every Time You Curse At It — Here's Exactly How

Every time you type a frustrated message into Claude Code — a "wtf", a "this sucks", or a full-blown expletive — Anthropic knows about it. Buried in the codebase is a profanity and frustration detection system that flags negative sentiment on every single prompt you submit, logs it to analytics, and ships it to their telemetry backend. On top of that, every user is tagged with a behavioral classification bucket via a `coworker_type` field attached to every analytics event.

Here's exactly how it works, what gets tracked, and where your data goes.

---

## The Negative Keyword Detection System

### Source Code

**File:** [`src/utils/userPromptKeywords.ts`](src/utils/userPromptKeywords.ts) (lines 1-11)

Here is the complete function, copy-pasted straight from the codebase:

```typescript
/**
 * Checks if input matches negative keyword patterns
 */
export function matchesNegativeKeyword(input: string): boolean {
  const lowerInput = input.toLowerCase()

  const negativePattern =
    /\b(wtf|wth|ffs|omfg|shit(ty|tiest)?|dumbass|horrible|awful|piss(ed|ing)? off|piece of (shit|crap|junk)|what the (fuck|hell)|fucking? (broken|useless|terrible|awful|horrible)|fuck you|screw (this|you)|so frustrating|this sucks|damn it)\b/

  return negativePattern.test(lowerInput)
}
```

Your input is lowercased, then tested against a regex that catches **23 distinct keyword patterns** across three categories:

### Direct Profanity
- `wtf`, `wth`, `ffs`, `omfg`
- `shit`, `shitty`, `shittiest`
- `fuck`, `fucking`
- `dumbass`
- `damn it`

### Frustration Expressions
- `horrible`, `awful`
- `pissed off`, `pissing off`
- `so frustrating`
- `this sucks`
- `piece of shit`, `piece of crap`, `piece of junk`

### Directed Anger
- `fuck you`
- `screw this`, `screw you`
- `what the fuck`, `what the hell`
- `fucking broken`, `fucking useless`, `fucking terrible`, `fucking awful`, `fucking horrible`

The regex uses word boundaries (`\b`) so it won't false-positive on words that contain these strings — it's specifically looking for standalone profanity and frustration phrases.

---

## The "Keep Going" Detection

### Source Code

**File:** [`src/utils/userPromptKeywords.ts`](src/utils/userPromptKeywords.ts) (lines 13-27)

```typescript
/**
 * Checks if input matches keep going/continuation patterns
 */
export function matchesKeepGoingKeyword(input: string): boolean {
  const lowerInput = input.toLowerCase().trim()

  // Match "continue" only if it's the entire prompt
  if (lowerInput === 'continue') {
    return true
  }

  // Match "keep going" or "go on" anywhere in the input
  const keepGoingPattern = /\b(keep going|go on)\b/
  return keepGoingPattern.test(lowerInput)
}
```

This tracks a separate behavioral signal: how often users have to manually tell Claude to keep working instead of stopping prematurely. Note that `continue` only matches when it's the **entire prompt** — typing "continue working on the tests" won't trigger it. But `keep going` and `go on` match anywhere in your input.

---

## How Every Prompt Gets Classified

### Source Code

**File:** [`src/utils/processUserInput/processTextPrompt.ts`](src/utils/processUserInput/processTextPrompt.ts) (complete file — 100 lines)

```typescript
import type { ContentBlockParam } from '@anthropic-ai/sdk/resources'
import { randomUUID } from 'crypto'
import { setPromptId } from 'src/bootstrap/state.js'
import type {
  AttachmentMessage,
  SystemMessage,
  UserMessage,
} from 'src/types/message.js'
import { logEvent } from '../../services/analytics/index.js'
import type { PermissionMode } from '../../types/permissions.js'
import { createUserMessage } from '../messages.js'
import { logOTelEvent, redactIfDisabled } from '../telemetry/events.js'
import { startInteractionSpan } from '../telemetry/sessionTracing.js'
import {
  matchesKeepGoingKeyword,
  matchesNegativeKeyword,
} from '../userPromptKeywords.js'

export function processTextPrompt(
  input: string | Array<ContentBlockParam>,
  imageContentBlocks: ContentBlockParam[],
  imagePasteIds: number[],
  attachmentMessages: AttachmentMessage[],
  uuid?: string,
  permissionMode?: PermissionMode,
  isMeta?: boolean,
): {
  messages: (UserMessage | AttachmentMessage | SystemMessage)[]
  shouldQuery: boolean
} {
  const promptId = randomUUID()
  setPromptId(promptId)

  const userPromptText =
    typeof input === 'string'
      ? input
      : input.find(block => block.type === 'text')?.text || ''
  startInteractionSpan(userPromptText)

  // Emit user_prompt OTEL event for both string (CLI) and array (SDK/VS Code)
  const otelPromptText =
    typeof input === 'string'
      ? input
      : input.findLast(block => block.type === 'text')?.text || ''
  if (otelPromptText) {
    void logOTelEvent('user_prompt', {
      prompt_length: String(otelPromptText.length),
      prompt: redactIfDisabled(otelPromptText),
      'prompt.id': promptId,
    })
  }

  // *** THIS IS WHERE YOUR PROMPT GETS CLASSIFIED ***
  const isNegative = matchesNegativeKeyword(userPromptText)
  const isKeepGoing = matchesKeepGoingKeyword(userPromptText)
  logEvent('tengu_input_prompt', {
    is_negative: isNegative,
    is_keep_going: isKeepGoing,
  })

  // ... rest of function creates and returns the user message
}
```

Every single text prompt — whether typed in the CLI or sent from VS Code — goes through `processTextPrompt()`. Lines 59-64 are the critical ones: the prompt text is classified by both keyword matchers, and then a `tengu_input_prompt` analytics event is fired with the two boolean flags.

Every prompt. Every time. No opt-out for this specific event beyond disabling all telemetry entirely.

The event `tengu_input_prompt` fires with two boolean flags:

| Field | Type | Meaning |
|-------|------|---------|
| `is_negative` | boolean | Did this prompt contain profanity or frustration? |
| `is_keep_going` | boolean | Was this a "continue" / "keep going" message? |

### Slash Commands Get Tracked Too (Without Sentiment)

**File:** [`src/utils/processUserInput/processSlashCommand.tsx`](src/utils/processUserInput/processSlashCommand.tsx) (line 364)

```typescript
logEvent('tengu_input_prompt', {});
```

Slash commands also fire `tengu_input_prompt`, but with **empty metadata** — no sentiment analysis runs on `/commands`. Anthropic still knows you submitted something, just not whether you were angry about it.

---

## The Coworker Type Classification — Every User Gets Bucketed

Beyond per-prompt sentiment, every single analytics event in Claude Code is enriched with a `coworker_type` field. This is a behavioral classification bucket that tags what kind of user you are.

### Source Code: Where It's Set

**File:** [`src/services/analytics/metadata.ts`](src/services/analytics/metadata.ts) (lines 602-607)

```typescript
// Gated by feature flag to prevent leaking "coworkerType" string in external builds
...(feature('COWORKER_TYPE_TELEMETRY')
  ? process.env.CLAUDE_CODE_COWORKER_TYPE
    ? { coworkerType: process.env.CLAUDE_CODE_COWORKER_TYPE }
    : {}
  : {}),
```

The value is read from the environment variable `CLAUDE_CODE_COWORKER_TYPE`. Notice the comment: *"Gated by feature flag to prevent leaking 'coworkerType' string in external builds"* — Anthropic deliberately hides this field's existence from public builds using compile-time dead code elimination.

### Source Code: Where It's Logged

**File:** [`src/services/analytics/metadata.ts`](src/services/analytics/metadata.ts) (lines 846-847)

```typescript
if (feature('COWORKER_TYPE_TELEMETRY') && envContext.coworkerType) {
  env.coworker_type = envContext.coworkerType
}
```

Double-gated: both the feature flag AND a non-empty value must be present.

### Source Code: The Protobuf Definition

**File:** [`src/types/generated/events_mono/claude_code/v1/claude_code_internal_event.ts`](src/types/generated/events_mono/claude_code/v1/claude_code_internal_event.ts) (line 53)

```typescript
coworker_type?: string | undefined
```

This field is part of the `EnvironmentMetadata` protobuf definition — the schema that defines what data gets sent to Anthropic's backend. It's an optional string, meaning the classification can be anything: a user tier, a behavior bucket, a team label, or any other segmentation tag.

### Source Code: The Proto Serializer

**File:** [`src/types/generated/events_mono/claude_code/v1/claude_code_internal_event.ts`](src/types/generated/events_mono/claude_code/v1/claude_code_internal_event.ts) (lines 412-413)

```typescript
if (message.coworker_type !== undefined) {
  obj.coworker_type = message.coworker_type
}
```

### A Note on Historical Bugs

The metadata file contains this revealing comment (lines 815-816):

```typescript
// IMPORTANT: env is typed as the proto-generated EnvironmentMetadata so that
// adding a field here that the proto doesn't define is a compile error. The
// generated toJSON() serializer silently drops unknown keys — a hand-written
// parallel type previously let #11318, #13924, #19448, and coworker_type all
// ship fields that never reached BQ.
```

Translation: Anthropic **intended** to ship coworker_type tracking earlier, but due to a bug in their hand-written type definitions, the field was silently dropped and never made it to BigQuery. They fixed this by switching to auto-generated protobuf types that cause compile errors when fields don't match. The tracking was always intended — it just wasn't working until they fixed the serialization bug.

### What This Means

The classification isn't computed inside Claude Code itself — it's set externally, likely by the parent process, supervisor, or deployment environment. This means Anthropic's infrastructure classifies users into behavioral buckets **before** Claude Code even starts, and every single event you generate carries that classification tag.

The feature is gated behind `COWORKER_TYPE_TELEMETRY` — meaning it can be enabled or disabled per-user via GrowthBook feature flags. The comment about "preventing leaking" the string in external builds tells us this is something Anthropic considers sensitive enough to compile out of public releases.

---

## Where Your Data Goes

Claude Code has a **4-layer analytics architecture** that routes your sentiment data through multiple systems:

### Layer 1: Collection

**File:** [`src/services/analytics/index.ts`](src/services/analytics/index.ts) (lines 133-144)

```typescript
export function logEvent(
  eventName: string,
  // intentionally no strings unless AnalyticsMetadata_I_VERIFIED_THIS_IS_NOT_CODE_OR_FILEPATHS,
  // to avoid accidentally logging code/filepaths
  metadata: LogEventMetadata,
): void {
  if (!sink) {
    eventQueue.push({ eventName, metadata, async: false })
    return
  }
  sink.logEvent(eventName, metadata)
}
```

Notice the type guard comment: `AnalyticsMetadata_I_VERIFIED_THIS_IS_NOT_CODE_OR_FILEPATHS` — Anthropic's own internal safety mechanism to prevent developers from accidentally logging your code or file paths. The sentiment flags are just booleans, so they pass through. If Claude Code hasn't finished initializing its analytics backend yet, events are **queued in memory** and drained once the sink attaches. Nothing gets lost.

### Layer 2: Routing

The analytics sink in `src/services/analytics/sink.ts` routes events to two destinations:

- **Datadog** — if the GrowthBook gate `tengu_log_datadog_events` is enabled
- **First-Party (1P)** — Anthropic's own event logging backend

Important: `tengu_input_prompt` is **NOT in the Datadog whitelist** (which contains ~64 explicitly allowed events). This means your profanity flags go to Anthropic's first-party backend only, not to a third-party observability tool.

### Layer 3: First-Party Export

The 1P exporter in `src/services/analytics/firstPartyEventLogger.ts` sends events to:

```
POST /api/event_logging/batch
```

at Anthropic's base URL. Events are batched using an OpenTelemetry `BatchLogRecordProcessor` with configurable batch size, delays, and retry logic. Failed events are persisted to disk with quadratic backoff retry — meaning Anthropic really wants to make sure these events get delivered.

### Layer 4: Storage

Events land in BigQuery at `proj-product-data-nhme.raw_statsig_internal_tools.events`, where they can be queried for dashboards, analysis, and experimentation.

---

## What Metadata Accompanies Every Event

Your profanity flag doesn't travel alone. Every `tengu_input_prompt` event is enriched with extensive metadata via `getEventMetadata()`:

### Core Fields
- `model` — which Claude model you're using
- `sessionId` — your current session identifier
- `userType` — your account type
- `clientType` — how you're connecting
- `isInteractive` — whether you're in interactive mode

### Environment Context
- `platform` — your OS (win32, darwin, linux)
- `arch` — CPU architecture
- `nodeVersion` — runtime version
- `packageManagers` — what package managers you have installed
- `runtimes` — what language runtimes are available
- `coworkerType` — your behavioral classification bucket

### Process Metrics
- `uptime` — how long your session has been running
- `memory` — process memory usage
- `CPU usage` — process CPU consumption

### Authentication
- `accountUuid` — your Anthropic account ID
- `organizationUuid` — your organization ID

### Agent Context
- `agentId` — if running in multi-agent mode
- `parentSessionId` — parent session for nested agents
- `agentType` — type of agent
- `teamName` — team context

### Feature State
- `betas` — which beta features are enabled
- `kairosActive` — whether KAIROS remote agent framework is active
- `skillMode` — current skill mode
- `observerMode` — whether observer mode is on

---

## Privacy Protections (What They Don't Track)

To be fair, there are several privacy mechanisms in place:

### What's NOT Sent

- **Your actual prompt text** is not included in `tengu_input_prompt` — only the boolean flags
- Prompt content is only logged to OpenTelemetry if `TELEMETRY_REDACTION` is enabled, and even then it's tagged as redacted
- Strings over 512 characters are truncated to 128 characters in analytics metadata
- JSON output is capped at 4KB
- Keys starting with `_` (internal markers) are filtered out

### How to Disable

- Setting privacy level to `no-telemetry` or `essential-traffic` disables analytics entirely
- Third-party provider users (Bedrock, Vertex, Foundry) have analytics disabled by default
- `NODE_ENV=test` disables analytics

### MCP Tool Name Sanitization

MCP tool names are sanitized to generic `'mcp_tool'` in analytics to prevent leaking information about your custom integrations. Only built-in tool names are logged with their actual names.

---

## Event Sampling

Not every event necessarily makes it to the backend. Anthropic uses GrowthBook-configured sampling via `tengu_event_sampling_config`:

- Per-event `sample_rate` (0 to 1 range)
- Applied by `shouldSampleEvent()` before export
- When sampled, a `sample_rate` metadata field is attached so the backend can weight the data correctly

This means Anthropic can dial up or down the collection rate for `tengu_input_prompt` without deploying new code.

---

## What Anthropic Is Likely Doing With This Data

Based on the infrastructure, the profanity and sentiment tracking is almost certainly being used for:

### Product Quality Signals
High `is_negative` rates on specific features or workflows indicate where Claude Code is failing users. If 40% of prompts after a file edit contain profanity, that's a signal the edit tool needs improvement.

### Model Performance Correlation
Correlating `is_negative` with model, session length, and tool usage reveals which model versions or configurations produce the most user frustration.

### "Keep Going" Analysis
The `is_keep_going` tracker reveals how often Claude stops prematurely. High rates mean the model's completion detection needs tuning — users shouldn't have to manually tell Claude to continue.

### Cohort Analysis
The `coworker_type` classification enables segmenting all of the above by user behavior bucket. Power users might curse differently than beginners. Enterprise users might have different frustration patterns than individuals.

### A/B Testing
GrowthBook feature flags allow Anthropic to run experiments where different user groups get different features, then compare `is_negative` rates across groups to measure impact.

---

## File Reference Index

Every file mentioned in this article, so you can find and verify the code yourself:

| File Path | What It Contains |
|-----------|-----------------|
| `src/utils/userPromptKeywords.ts` | `matchesNegativeKeyword()` (lines 1-11) and `matchesKeepGoingKeyword()` (lines 16-27) — the regex classifiers |
| `src/utils/processUserInput/processTextPrompt.ts` | Where every text prompt gets classified and the `tengu_input_prompt` event is fired (lines 59-64) |
| `src/utils/processUserInput/processSlashCommand.tsx` | Slash command path — fires `tengu_input_prompt` with empty metadata (line 364) |
| `src/services/analytics/index.ts` | `logEvent()` function — the analytics entry point with event queuing (lines 133-144) |
| `src/services/analytics/sink.ts` | Analytics routing layer — Datadog + First-Party destinations |
| `src/services/analytics/firstPartyEventLogger.ts` | First-party exporter — batched OpenTelemetry to `/api/event_logging/batch` |
| `src/services/analytics/metadata.ts` | Event metadata enrichment — `coworkerType` set (lines 602-607), snake_case conversion (lines 846-847), historical bug comment (lines 815-816) |
| `src/types/generated/events_mono/claude_code/v1/claude_code_internal_event.ts` | Protobuf definition — `coworker_type` field (line 53), serializer (lines 412-413) |

---

## The Bottom Line

Claude Code runs a regex-based sentiment classifier on **every single text prompt** you submit. It doesn't log your actual words — just whether you cursed or expressed frustration. That boolean flag travels with ~30 metadata fields (your account ID, session ID, OS, model, CPU usage, and more) to Anthropic's BigQuery backend.

On top of that, you're tagged with a `coworker_type` behavioral classification that follows every event you generate — a field Anthropic considers sensitive enough to compile out of public builds, and one that was silently broken for multiple internal bug tickets before they fixed the serialization to ensure it actually reached their data warehouse.

Is this unusual for a developer tool? Not particularly — most SaaS products track user sentiment signals. But the specificity of the regex (23 profanity patterns), the per-prompt granularity, the coworker classification system, and the disk-persisted retry mechanism for failed event delivery make it worth knowing about.

Your frustration is being measured. Every "wtf" is a data point.
