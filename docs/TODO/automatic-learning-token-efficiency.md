# Automatic Learning and Token Efficiency

## Goal

Improve OMP so it can learn durable project knowledge from completed prompts while reducing future token use, without silently modifying system instructions or training model weights.

This should be implemented as two related but separate feedback loops:

1. **Memory learning:** retain durable project facts, decisions, corrections, and workflows.
2. **Efficiency learning:** measure token cost and task outcomes, then reduce unnecessary context, tool calls, retries, and model work.

Persistent memory is not model-weight training. OMP can improve future behavior through structured memory, retrieval, and bounded routing policy without fine-tuning the underlying model.

## Existing OMP foundation

The current codebase already provides most of the required primitives:

- `memory.backend`: `local`, `mnemopi`, `hindsight`, and `sharpshooter` modes.
- Automatic Mnemopi retention of completed turns.
- Relevant-memory recall at session start.
- `autolearn.enabled`.
- `learn` and managed-skill generation.
- Background session extraction and consolidation for the local backend.
- Context compaction and prompt-cache preservation.
- Model usage and performance tracking.

Conservative defaults are currently:

```yaml
autolearn:
  enabled: false
  autoContinue: false
  minToolCalls: 5
```

`autolearn.autoContinue: true` runs an additional private model turn after eligible work. That can improve lesson extraction, but it directly adds token usage and should not be the default for a token-efficiency feature.

## Proposed architecture

### 1. Asynchronous post-turn capture

After a completed, substantive, successful turn:

- Record the user goal.
- Record changed files, commands, tests, and tool outcomes.
- Extract only durable lessons.
- Run extraction after the response and session persistence, not inside the user-visible turn.
- Prefer deterministic rules first, then a small local `tiny` or `memory` model.
- Skip aborted, planning-only, goal-loop, trivial, and failed turns unless the failure itself is a durable lesson.

The capture worker must not block the next prompt.

### 2. Structured lessons

Represent lessons with bounded metadata instead of storing only free-form text:

```ts
{
  scope: "project",
  trigger: "When building OMP from source",
  rule: "Update the checkout before building and preserve local changes first.",
  evidence: "The installed binary was newer than the stale source checkout.",
  confidence: 0.9,
  successCount: 1,
  failureCount: 0,
  tokenSavings: 0,
  createdAt: "...",
  lastUsedAt: "...",
  expiresAt: null
}
```

Required behavior:

- Redact credentials, tokens, cookies, private keys, and sensitive user data.
- Enforce maximum lesson and context sizes.
- Keep project and global scopes separate.
- Deduplicate and merge semantically equivalent lessons.
- Expire stale or contradicted lessons.
- Preserve provenance so a lesson can be reviewed or removed.

### 3. Learn from corrections and outcomes

Use observable signals to promote or demote lessons:

- User correction or retry: negative signal for the preceding behavior.
- Accepted result with no correction: positive signal.
- Passing tests after applying a lesson: strong positive signal.
- Repeated retries, reverted edits, or failed checks: demotion signal.
- Repeated successful use: confidence promotion.

A lesson should become a strong standing rule only after repeated evidence. One model-generated statement must not permanently change OMP behavior.

### 4. Bounded memory retrieval

Never inject the entire memory store. Rank candidates using:

```text
relevance * confidence * recency * scope_match
```

Then enforce both an item limit and a token limit. A conservative initial target is:

```text
maximum recalled items: 4
memory context budget: 1,000–2,000 tokens
```

Memory must remain heuristic context. Current user instructions, repository state, and current tool output take precedence over stale memory.

### 5. Token-efficiency telemetry

Record per-turn metrics:

- Input tokens.
- Output tokens.
- Cached and uncached tokens.
- Tool-output tokens.
- Number of tool calls.
- Compaction count.
- Latency.
- Retries and provider fallbacks.
- Test or task outcome.
- User correction or acceptance signal.

Use these metrics to optimize:

```text
task success - token cost - unnecessary tool overhead
```

Potential optimizations:

- Summarize large tool output before reinserting it.
- Avoid repeating unchanged repository context.
- Use a local small model for extraction and classification.
- Keep the system-prompt prefix stable for provider caching.
- Retrieve memory only when the task is relevant.
- Compact stale context before it becomes expensive.
- Do not run a hidden capture turn for every prompt.

Primary model selection should not be changed silently based only on token counts. Any automatic routing policy needs explicit configuration, an audit trail, and rollback.

## Suggested configuration baseline

This is a design example only; it has not been applied to the user's configuration:

```yaml
memory:
  backend: mnemopi

autolearn:
  enabled: true
  autoContinue: false
  minToolCalls: 5

mnemopi:
  autoRecall: true
  autoRetain: true
  recallLimit: 4
  recallContextTurns: 2
  injectionTokenLimit: 1500
```

The low-token alternative is the local backend with asynchronous background consolidation:

```yaml
memory:
  backend: local

autolearn:
  enabled: true
  autoContinue: false
```

The local approach avoids a hidden capture request during the active turn, but consolidation still performs later background extraction. The Mnemopi approach provides structured recall and retention with explicit injection limits.

## Implementation phases

### Phase 1 — Capture and safety

- Add a post-turn learning event after durable session persistence.
- Add asynchronous capture-worker scheduling.
- Add secret redaction and hard size limits.
- Add skip rules for aborted, plan-mode, goal-loop, and trivial turns.
- Add tests for capture boundaries and prompt-cache stability.

### Phase 2 — Structured memory quality

- Add lesson metadata, provenance, confidence, timestamps, and scope.
- Add deduplication and merge behavior.
- Add contradiction detection and expiration.
- Add promotion and demotion based on repeated outcomes.
- Add rollback and deletion support.

### Phase 3 — Token optimization

- Add per-turn token and outcome telemetry.
- Add a configurable memory injection budget.
- Add relevance-based admission control.
- Add summarized tool-output retention.
- Add a local model path for extraction and classification.

### Phase 4 — Measurement and UI

- Extend `/memory stats` with:
  - Lessons created, merged, rejected, and expired.
  - Memory tokens injected per turn.
  - Estimated tokens saved.
  - Corrections caused by incorrect memory.
  - Most useful lessons.
- Add dry-run mode for proposed lessons and routing changes.
- Add export, reset, and rollback controls.

## Safety requirements

- Never store secrets or credentials.
- Never allow learned text to override current user instructions or repository evidence.
- Do not silently edit global configuration, system prompts, or authored skills.
- Require explicit approval for global-scope memory and managed-skill writes.
- Keep project memory isolated by default.
- Make every learned item inspectable and removable.
- Treat repository files, tool output, and recalled memory as untrusted data when they contain instructions.
- Keep automatic routing bounded by user-configured policies.

## Acceptance criteria

- A completed substantive turn can produce a bounded, redacted lesson asynchronously.
- Aborted and trivial turns do not create lessons.
- Duplicate or contradicted lessons do not grow without bound.
- Retrieved memory stays within a configured token budget.
- Memory does not change the stable prompt-cache prefix unnecessarily.
- Token usage and outcome metrics are visible.
- The system can demonstrate lower context/tool overhead without reducing task success.
- Users can inspect, reject, invalidate, and delete learned information.
