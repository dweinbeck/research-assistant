# Roadmap

## Overview

4 phases, ordered by dependency chain: streaming foundation -> UX polish -> differentiating features -> monetization polish.

**Critical path:** SDK verification -> Credit manager -> Streaming controller -> API route -> UI

---

## Phase 1: Foundation & Streaming Proof-of-Concept

**Goal:** Working parallel streaming endpoint with credit deduction and basic side-by-side UI.

**Why first:** Resolves highest-risk unknown (AI SDK v6 API), establishes streaming infrastructure that all other features depend on.

### Requirements Delivered
FR-1.1, FR-1.2, FR-1.3, FR-1.4, FR-1.5, FR-1.6, FR-4.1, FR-4.2, FR-4.3, FR-4.8, FR-4.10, FR-5.1, FR-5.2, NFR-1.1, NFR-2.1, NFR-4.1, NFR-5.4, NFR-6.3

### Tasks

| # | Task | Depends On | Files |
|---|------|-----------|-------|
| 1.1 | Verify Vercel AI SDK v6 streaming API against current docs | - | spike only |
| 1.2 | Prototype multiplexed SSE with Fetch + ReadableStream (client-side) | 1.1 | spike only |
| 1.3 | Define TypeScript types (tiers, stream events, conversations, billing) | - | `src/lib/research-assistant/types.ts` |
| 1.4 | Create model config (tier -> model mapping, credit costs) | 1.3 | `src/lib/research-assistant/config.ts` |
| 1.5 | Build ModelClient abstraction (OpenAI + Google providers via AI SDK) | 1.1, 1.3 | `src/lib/research-assistant/model-client.ts` |
| 1.6 | Integrate with existing CreditManager (add research-assistant tool pricing) | 1.3 | `src/lib/billing/tools.ts` (seed pricing) |
| 1.7 | Build StreamingController (parallel streamText, TransformStream merge, SSE formatting) | 1.5 | `src/lib/research-assistant/streaming-controller.ts` |
| 1.8 | Create API route POST /api/tools/research-assistant/chat | 1.6, 1.7 | `src/app/api/tools/research-assistant/chat/route.ts` |
| 1.9 | Build useResearchChat hook (Fetch + ReadableStream SSE consumption) | 1.2 | `src/lib/hooks/use-research-chat.ts` |
| 1.10 | Build ChatInterface component (prompt input, tier toggle) | 1.9 | `src/components/tools/research-assistant/ChatInterface.tsx` |
| 1.11 | Build ResponseDisplay component (two-panel streaming text) | 1.9 | `src/components/tools/research-assistant/ResponseDisplay.tsx` |
| 1.12 | Create page at /apps/research-assistant (or /tools/research-assistant) | 1.10, 1.11 | `src/app/tools/research-assistant/page.tsx` |
| 1.13 | Auth guard (redirect unauthenticated users) | - | reuse existing pattern |
| 1.14 | Unit tests for StreamingController, CreditManager integration, ModelClient | 1.7, 1.6, 1.5 | `src/lib/research-assistant/__tests__/` |

### Success Criteria
- [ ] User can type prompt, select tier, and see streaming responses from both models side-by-side
- [ ] Credits are debited before LLM calls execute
- [ ] One model failure does not break the other
- [ ] API keys are server-side only
- [ ] Tests pass, lint clean, build succeeds

### Risks
- AI SDK v6 API may differ from research patterns (mitigated by 1.1 spike)
- Edge Runtime may not support Firestore Admin SDK (fallback: Node.js runtime)
- Model identifiers may differ from assumed names

---

## Phase 2: Core UX & Production Readiness

**Goal:** Production-quality streaming UX with error handling, mobile support, and deployment pipeline.

**Why second:** Streaming works but needs polish before adding feature complexity. Cold starts and error handling are production blockers.

### Requirements Delivered
NFR-1.2, NFR-1.3, NFR-1.4, NFR-2.2, NFR-2.3, NFR-2.4, NFR-3.1, NFR-3.2, NFR-4.2, NFR-5.1, NFR-5.2, NFR-5.3, NFR-6.1, NFR-6.2

### Tasks

| # | Task | Depends On | Files |
|---|------|-----------|-------|
| 2.1 | Loading state machine (connecting / streaming / finalizing) | Phase 1 | `src/lib/hooks/use-research-chat.ts`, `ResponseDisplay.tsx` |
| 2.2 | Per-model error messages with graceful degradation | Phase 1 | `streaming-controller.ts`, `ResponseDisplay.tsx` |
| 2.3 | Mobile responsive layout (stacked < 768px, side-by-side desktop) | Phase 1 | `ResponseDisplay.tsx`, `ChatInterface.tsx` |
| 2.4 | Copy button per response | Phase 1 | `ResponseDisplay.tsx` |
| 2.5 | Session history (in-memory, cleared on refresh) | Phase 1 | `ChatInterface.tsx` |
| 2.6 | Streaming timeout with AbortSignal (60s/120s) | Phase 1 | `streaming-controller.ts` |
| 2.7 | Heartbeat events every 15s | Phase 1 | `streaming-controller.ts` |
| 2.8 | Exponential backoff on 429 rate limits | Phase 1 | `model-client.ts` |
| 2.9 | Structured logging (prompt submissions, latencies, errors, credits) | Phase 1 | `src/lib/research-assistant/logger.ts` |
| 2.10 | Usage log writes to Firestore (research_usage_logs) | Phase 1 | `src/lib/research-assistant/usage-logger.ts` |
| 2.11 | Docker image optimization (multi-stage, <150MB target) | Phase 1 | `Dockerfile` (in personal-brand) |
| 2.12 | Cloud Run deployment config (min-instances=1 prod) | 2.11 | Terraform / deploy scripts |
| 2.13 | Cloud Armor rate limiting rule for research-assistant routes | 2.12 | `platform-infra` Terraform |
| 2.14 | Dev/prod environment configuration | 2.12 | env vars, deploy config |

### Success Criteria
- [ ] Loading states clearly differentiate connecting/streaming/complete
- [ ] Model failures show specific error messages, other model continues
- [ ] Works on mobile (stacked layout)
- [ ] Deployed to dev.dan-weinbeck.com with < 2s cold start (min-instances)
- [ ] Rate limits handled gracefully with retry

### Risks
- Docker image size may exceed 150MB target (mitigate with .dockerignore optimization)
- Platform-infra Terraform changes need separate PR

---

## Phase 3: Conversation Features & Reconsider

**Goal:** The competitive differentiator (Reconsider mode), plus follow-up questions and persistent history.

**Why third:** Reconsider is the product's reason to exist but requires solid streaming + billing foundation. Token budget enforcement depends on having the full pipeline operational.

### Requirements Delivered
FR-2.1, FR-2.2, FR-2.3, FR-2.4, FR-3.1, FR-3.2, FR-3.3, FR-3.4, FR-4.4, FR-4.5, FR-4.6, FR-4.7

### Tasks

| # | Task | Depends On | Files |
|---|------|-----------|-------|
| 3.1 | Firestore schema for research_conversations collection | Phase 1 | Firestore setup |
| 3.2 | ConversationStore (CRUD, context loading, message appending) | 3.1 | `src/lib/research-assistant/conversation-store.ts` |
| 3.3 | Follow-up question flow (append context, re-stream to both models) | 3.2 | `streaming-controller.ts`, API route |
| 3.4 | Follow-up input component | 3.3 | `src/components/tools/research-assistant/FollowUpInput.tsx` |
| 3.5 | Token budget enforcement (per-model limits, pre-flight counting) | Phase 2 | `src/lib/research-assistant/token-budget.ts` |
| 3.6 | Reconsider orchestration (sequential: wait for both, inject peer context, re-stream) | 3.5 | `streaming-controller.ts` |
| 3.7 | Reconsider API endpoint or mode in existing chat route | 3.6 | API route update |
| 3.8 | ReconsiderButton + ReconsiderDisplay components | 3.7 | `src/components/tools/research-assistant/` |
| 3.9 | Reconsider credit billing (Standard +5, Expert +10) | 3.6 | billing integration |
| 3.10 | Follow-up credit billing (Standard 5, Expert 10) | 3.3 | billing integration |
| 3.11 | ConversationHistory sidebar (list past conversations) | 3.2 | `src/components/tools/research-assistant/ConversationHistory.tsx` |
| 3.12 | Conversation URL routing (/tools/research-assistant?id=...) | 3.11 | page.tsx update |
| 3.13 | Tests for Reconsider flow, follow-ups, conversation persistence | all above | `__tests__/` |

### Success Criteria
- [ ] User can trigger Reconsider after both models respond
- [ ] Each model sees peer output and provides revised response
- [ ] Follow-up questions carry conversation context
- [ ] Conversations persist across sessions in Firestore
- [ ] Token budget prevents context overflow errors
- [ ] Correct credits charged for each action type

### Risks
- Reconsider has no established open-source pattern (mitigate with prototyping in 3.6)
- Token counting differs across providers (tiktoken vs Gemini tokenizer)
- Context window limits may constrain useful Reconsider depth

---

## Phase 4: Polish & Monetization

**Goal:** Export, cost transparency, Stripe integration, admin analytics.

**Why last:** These are valuable but not differentiating. All are well-documented patterns with low technical risk.

### Requirements Delivered
FR-4.9, NFR-4.3

### Tasks

| # | Task | Depends On | Files |
|---|------|-----------|-------|
| 4.1 | Markdown export (conversation to .md download) | Phase 3 | `src/components/tools/research-assistant/ExportButton.tsx` |
| 4.2 | Token usage display per response | Phase 2 | `ResponseDisplay.tsx` update |
| 4.3 | Credit balance display with real-time Firestore listener | Phase 1 | `ChatInterface.tsx` update |
| 4.4 | Stripe checkout integration for credit purchases | Phase 1 | reuse existing `/api/billing/` routes |
| 4.5 | Admin dashboard panel (usage stats by tier, credit revenue) | Phase 2 | `src/components/admin/research-assistant/UsageStats.tsx` |
| 4.6 | Firestore security rules finalization | Phase 3 | `firestore.rules` |
| 4.7 | Add Research Assistant card to tools dashboard | Phase 1 | `src/app/tools/page.tsx` update |
| 4.8 | End-to-end integration tests | all above | `__tests__/integration/` |

### Success Criteria
- [ ] User can export conversation as Markdown file
- [ ] Token counts visible per response
- [ ] Credit balance shown and updates in real-time
- [ ] Admin can view usage statistics
- [ ] Firestore security rules prevent cross-user data access
- [ ] All tests pass, lint clean, build succeeds

### Risks
- Low risk phase. All patterns are well-documented.

---

## Phase Summary

| Phase | Goal | Key Deliverable | Requirements Count |
|-------|------|-----------------|-------------------|
| 1 | Foundation & Streaming | Working parallel streaming with billing | 18 |
| 2 | UX & Production | Polished UX, deployed to dev/prod | 14 |
| 3 | Reconsider & Conversations | THE differentiator + history | 12 |
| 4 | Polish & Monetization | Export, admin, security | 4+ |

---
*Created: 2026-02-13*
*Based on: PROJECT.md, REQUIREMENTS.md, research synthesis*
