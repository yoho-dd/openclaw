# Concurrent Messaging with Intelligent Interruption

## Overview

Implement a new queue mode `"smart-interrupt"` that enables human-like conversational flow on IM channels. When the user sends new messages while the agent is busy, the system:

1. **Immediately acknowledges** the new message(s)
2. **Uses an LLM call to evaluate** whether the current task is still relevant
3. **Gracefully interrupts** at the nearest natural break point if the task is no longer relevant
4. **Re-plans** with combined context from old partial work + new messages

Initial rollout: Feishu first (via extension config), expandable to all channels.

## Architecture

```
User sends msg while agent is busy
    ↓
InboundDebouncer batches semantically-related messages (existing)
    ↓
get-reply-run.ts detects mode="smart-interrupt" + active run
    ↓
NEW: SmartInterruptEvaluator
    ├─ Sends brief ack to user ("收到，正在处理...")
    ├─ Makes fast LLM call to classify: "continue" | "interrupt" | "append"
    ├─ If "continue": enqueue as followup (like "followup" mode)
    ├─ If "append": steer into active session (like "steer" mode)
    └─ If "interrupt": set interrupt flag → graceful abort at next yield point
```

## Implementation Steps

### Step 1: Add `"smart-interrupt"` queue mode type

**File: `src/auto-reply/reply/queue/types.ts`**
- Add `"smart-interrupt"` to `QueueMode` union type

**File: `src/auto-reply/reply/queue/normalize.ts`**
- Add normalization for `"smart-interrupt"` / `"smartinterrupt"` / `"smart_interrupt"`

**File: `src/auto-reply/reply/queue/settings.ts`**
- No changes needed (generic mode resolution already works)

### Step 2: Create the smart-interrupt evaluator

**New file: `src/auto-reply/reply/smart-interrupt-evaluator.ts`** (~120 LOC)

```typescript
export type SmartInterruptDecision = "continue" | "interrupt" | "append";

export async function evaluateSmartInterrupt(params: {
  currentTaskSummary: string;    // What the agent is currently doing
  newMessages: string[];         // The new message(s) from the user
  sessionContext?: string;       // Recent conversation context
  signal?: AbortSignal;
}): Promise<SmartInterruptDecision>;
```

- Uses a fast/small model call (e.g. haiku or the session's configured model with low max_tokens)
- System prompt classifies the relationship between current task and new messages:
  - `"continue"`: New messages don't change the task direction (e.g., "thanks", "ok")
  - `"append"`: New messages add to the current task (e.g., "also add X", "and make it blue")
  - `"interrupt"`: New messages represent a task direction change (e.g., "actually, do Y instead", a new question)
- Timeout: 3s max, defaults to `"interrupt"` on timeout (fail-safe: prefer responsiveness)

### Step 3: Create the acknowledgment sender

**New file: `src/auto-reply/reply/smart-interrupt-ack.ts`** (~60 LOC)

```typescript
export async function sendSmartInterruptAck(params: {
  opts: GetReplyOptions;
  typing: TypingController;
  channel?: string;
  locale?: string;
}): Promise<void>;
```

- Sends a brief message like "Got it, re-evaluating..." / "收到，正在重新规划..."
- Locale-aware (detect from session context or channel config)
- Uses existing `opts.onBlockReply` or `opts.sendReply` to deliver

### Step 4: Wire into the agent-runner flow

**File: `src/auto-reply/reply/get-reply-run.ts`**
- After the existing `interrupt` mode block (line ~448), add a parallel block for `smart-interrupt`:

```typescript
if (resolvedQueue.mode === "smart-interrupt" && isActive) {
  // Collect current task summary from the active run's buffered text
  const currentTaskSummary = getEmbeddedPiRunBufferedText(sessionIdFinal) ?? "";

  // Send ack immediately
  await sendSmartInterruptAck({ opts, typing, channel: sessionCtx.Provider });

  // Evaluate relevance
  const decision = await evaluateSmartInterrupt({
    currentTaskSummary,
    newMessages: [baseBodyTrimmedRaw],
    signal: AbortSignal.timeout(3000),
  });

  // Route based on decision
  // (set resolvedQueue.mode override for downstream handling)
}
```

**File: `src/auto-reply/reply/agent-runner.ts`**
- In `runReplyAgent()`, handle the `smart-interrupt` mode:
  - If decision is `"append"` and streaming → steer (like existing steer path, line 197-204)
  - If decision is `"continue"` → enqueue as followup
  - If decision is `"interrupt"` → graceful abort + re-run

**File: `src/auto-reply/reply/queue-policy.ts`**
- Update `resolveActiveRunQueueAction()` to handle `smart-interrupt`:
  - When active: return `"evaluate-smart-interrupt"` (new action)
  - The caller handles the evaluation

### Step 5: Implement graceful abort (vs hard abort)

**File: `src/agents/pi-embedded-runner/runs.ts`**
- Add `requestGracefulAbort(sessionId)` function:
  - Sets a flag on the active run handle
  - Does NOT immediately call `AbortController.abort()`
  - Instead, the flag is checked at the next yield point (between tool calls)

**File: `src/agents/pi-embedded-runner/run/attempt.ts`**
- In the tool-call loop, after each tool result is processed, check the graceful abort flag:

```typescript
if (queueHandle.isGracefulAbortRequested()) {
  // Save partial work context
  const partialContext = buildPartialWorkContext(assistantTexts, toolMetas);
  // Inject a yield interrupt with context about why we're stopping
  await activeSession.steer(
    "[System: User sent new messages that change the task direction. " +
    "Wrap up current work at the nearest natural point and yield.]"
  );
}
```

- This allows the LLM to gracefully conclude rather than hard-cutting mid-response

### Step 6: Partial work preservation

**New file: `src/auto-reply/reply/smart-interrupt-context.ts`** (~80 LOC)

```typescript
export function buildInterruptedTaskContext(params: {
  partialAssistantText?: string;
  toolCallsMade?: string[];
  originalPrompt: string;
  interruptReason: string;
}): string;
```

- When the agent yields after graceful abort, builds a context note that gets prepended to the next turn
- Example: "Note: The previous task was interrupted because you sent new instructions. Partial progress: [summary]. Now responding to your latest messages."

### Step 7: Feishu-specific integration

**File: `extensions/feishu/` (or wherever Feishu channel config lives)**
- Set default queue mode to `"smart-interrupt"` for Feishu channel
- Or add to config docs: `messages.queue.byChannel.feishu: "smart-interrupt"`

### Step 8: Config and documentation

**File: `src/config/schema.help.ts`**
- Add help text for `smart-interrupt` mode

**File: `src/auto-reply/reply/directive-handling.queue-validation.ts`**
- Add `smart-interrupt` to valid modes list and help text

**File: `src/auto-reply/commands-registry.data.ts`**
- Add `smart-interrupt` to queue mode choices

### Step 9: Tests

**New file: `src/auto-reply/reply/smart-interrupt-evaluator.test.ts`**
- Test classification logic with various message scenarios

**File: `src/auto-reply/reply/agent-runner.misc.runreplyagent.test.ts`**
- Add test cases for smart-interrupt mode: continue/append/interrupt paths

**File: `src/auto-reply/reply/queue-policy.test.ts`**
- Add smart-interrupt to queue policy tests

## Key Design Decisions

1. **LLM evaluation over heuristics**: More accurate at understanding semantic intent. The 1-2s latency is acceptable because the user already sees an immediate ack message.

2. **Three-way classification** (continue/append/interrupt): Covers all cases:
   - "continue" = don't change anything, just queue
   - "append" = inject into current context (steer)
   - "interrupt" = stop and re-plan

3. **Graceful abort via steer, not hard abort**: Instead of killing the AbortController, we steer a "wrap up" instruction. The LLM can finish its current thought, save state, and yield naturally. Only if it doesn't yield within a timeout (e.g., 10s) do we hard-abort.

4. **Ack message**: Sent immediately before the LLM evaluation, so user sees instant feedback.

5. **Feishu-first**: Reduces risk. Once validated, `byChannel` config makes it easy to enable for other channels.

## Files to Create/Modify

| File | Action | ~LOC |
|------|--------|------|
| `src/auto-reply/reply/queue/types.ts` | Modify | +1 |
| `src/auto-reply/reply/queue/normalize.ts` | Modify | +5 |
| `src/auto-reply/reply/smart-interrupt-evaluator.ts` | **Create** | ~120 |
| `src/auto-reply/reply/smart-interrupt-ack.ts` | **Create** | ~60 |
| `src/auto-reply/reply/smart-interrupt-context.ts` | **Create** | ~80 |
| `src/auto-reply/reply/get-reply-run.ts` | Modify | +30 |
| `src/auto-reply/reply/agent-runner.ts` | Modify | +25 |
| `src/auto-reply/reply/queue-policy.ts` | Modify | +5 |
| `src/agents/pi-embedded-runner/runs.ts` | Modify | +20 |
| `src/agents/pi-embedded-runner/run/attempt.ts` | Modify | +15 |
| `src/config/schema.help.ts` | Modify | +2 |
| `src/auto-reply/reply/directive-handling.queue-validation.ts` | Modify | +2 |
| `src/auto-reply/commands-registry.data.ts` | Modify | +2 |
| `src/auto-reply/reply/smart-interrupt-evaluator.test.ts` | **Create** | ~150 |

**Total new code: ~370 LOC across 3 new files + ~100 LOC modifications across 10 existing files**
