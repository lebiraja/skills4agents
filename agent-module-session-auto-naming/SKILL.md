---
name: agent-module-session-auto-naming
description: "Module: Auto Session Naming Lifecycle"
---
# Module: Auto Session Naming Lifecycle

## 1. Module Metadata
- Module ID: `agent.module.session-auto-naming`
- Version: `1.0.0`
- Maturity: `production`
- Scope: Backend and frontend coordination for deterministic, non-blocking chat session title generation.
- Primary outcomes:
  - Replace placeholder title (`New Chat`) after first full exchange.
  - Never block response delivery for title generation.
  - Guarantee fallback title persistence when LLM generation fails.

## 2. Mission and Applicability
Use this module when building conversational systems that start with generic session names and must auto-rename sessions reliably.

Apply this module if all are true:
- Sessions are created before enough semantic context exists for naming.
- A first user message and first assistant response are available.
- UX requires immediate response rendering and delayed title refinement.

Do not apply as-is if:
- Naming must happen before assistant response is sent.
- A synchronous transactional rename is required for compliance reasons.

## 3. Architecture Pattern
- Pattern: `Deferred naming with deterministic fallback`
- Core design rules:
  - Initial title is always placeholder.
  - Naming trigger is first full exchange only.
  - Naming executes asynchronously.
  - Cleanup and truncation are deterministic.
  - Fallback derives from first user message.

Reference implementation boundaries:
- Backend session service: `backend/services/session_service.py`
- Backend session router: `backend/routers/sessions.py`
- Frontend session API client: `src/api/sessions.ts`
- Frontend state store: `src/store/sessionStore.ts`
- Chat UI flow: `src/components/ChatInterface.tsx`
- Session list UI: `src/components/SessionSidebar.tsx`

## 4. Implementation Workflow

### Phase A: Session Initialization
1. Create session with title `New Chat` and empty message list.
2. Persist `metadata.total_messages = 0`.
3. Return `session_id` and placeholder title immediately.

Exit criteria:
- Session exists with valid ID.
- Placeholder title visible in sidebar/list.

### Phase B: First Exchange Capture
1. Persist user message.
2. Generate assistant response.
3. Persist assistant message.
4. Evaluate naming trigger from pre-send snapshot.

Trigger condition:
- Run auto-title only when pre-exchange `metadata.total_messages == 0`.

Exit criteria:
- First exchange persisted.
- Background naming task scheduled once.

### Phase C: Background Title Generation
1. Build prompt from first user message and first assistant response snippet.
2. Request concise title from LLM.
3. Normalize output using deterministic rules.
4. Persist final title.

Normalization rules:
- Strip prefixes like `Title:` and surrounding quotes.
- Cap at 4 words and 50 characters.
- If empty or invalid, use first 4 user-message words.

Exit criteria:
- Session title is no longer placeholder.
- Fallback path persists valid title on generation failure.

### Phase D: Frontend Consistency Refresh
1. Return chat response immediately; do not await title task.
2. Refresh session list after first assistant response or through event channel.
3. Update displayed title when backend rename is detected.

Exit criteria:
- User sees renamed title without manual page reload.

## 5. Decision Framework
Use this matrix when implementing variants.

| Decision Area | Preferred Option | Alternative | Selection Rule |
|---|---|---|---|
| Rename execution | Async background task | Synchronous rename | Use async unless strict immediate consistency is mandated. |
| Rename trigger | First full exchange | First user message only | Use full exchange to maximize semantic quality. |
| Frontend update | Event-based push (SSE/WebSocket) | Timed refetch | Use push where infra supports persistent channels. |
| Fallback source | First user message words | Static timestamp-based label | Use user text for relevance and discoverability. |

## 6. Quality Gates and Validation Strategy

### Functional Validation
- Unit tests:
  - title cleanup and truncation behavior
  - invalid/empty output fallback behavior
- Integration tests:
  - rename runs exactly once on first exchange
  - subsequent exchanges never retrigger rename
  - manual rename endpoint contract matches service signature
- UI tests:
  - sidebar transitions `New Chat` -> generated title without full reload

### Failure Validation
- Simulate LLM timeout/error and verify fallback persistence.
- Simulate DB write failure and verify retry/log strategy.

### Contract Validation
- Ensure API payloads include fields needed for client-side optimistic update or follow-up fetch.

## 7. Benchmarks and SLO Targets
- Title generation success rate (including fallback): `>= 99.9%`
- Added latency to primary chat response due to naming: `0 ms` (non-blocking requirement)
- Background rename completion p95: `<= 2.0 s`
- Placeholder title stale rate after first exchange (within 5 s): `<= 1%`

## 8. Common Risks and Controls
- Risk: Endpoint-service argument mismatch.
  - Control: Contract test for every public naming endpoint.
- Risk: Sidebar never refreshes.
  - Control: Post-first-response refetch or server push event.
- Risk: Short low-quality title slips through.
  - Control: Enforce minimum semantic checks in normalization layer.

## 9. Agent Execution Checklist
- [ ] Session created with placeholder title and zero-message metadata.
- [ ] First exchange persisted before naming.
- [ ] Naming trigger guarded to run once.
- [ ] Background task non-blocking by design.
- [ ] Deterministic cleanup and fallback implemented.
- [ ] Frontend update path implemented and tested.
- [ ] Metrics and logs instrumented for rename pipeline.

## 10. Reuse Notes
This module is reusable for any chat-based product. Adapt only:
- Prompt strategy for domain-specific naming tone.
- Word/length limits by UX constraints.
- Frontend refresh mechanism by transport capability.
