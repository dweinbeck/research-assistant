# Project Research Summary

**Project:** Research Assistant (Multi-Model LLM Comparison Tool)
**Domain:** LLM Streaming Comparison -- Feature of Existing Personal Brand Site
**Researched:** 2026-02-13
**Confidence:** MEDIUM

## Executive Summary

The Research Assistant is a **new tool being added to an existing Next.js 16 personal brand site**, not a standalone application. It compares outputs from two LLM providers (OpenAI and Google) side-by-side in real-time using parallel streaming, with a unique "Reconsider" mode where models critique each other's outputs. The stack is pre-determined (Next.js 16, React 19, Tailwind 4, Firebase, Stripe, Cloud Run), so the core technical challenge is **parallel stream multiplexing** -- merging two independent LLM streams into a single SSE connection for real-time side-by-side rendering.

The recommended approach is to build in four phases, starting with the non-streaming foundation (Firestore schema, credit manager, model client abstraction), then layering parallel streaming on top, followed by conversation features and the Reconsider differentiator. This ordering is driven by a clear dependency chain: credit management must be race-condition-safe before any billable streaming occurs, streaming infrastructure must work before UI polish matters, and Reconsider requires token budget enforcement that depends on having the full streaming pipeline operational.

The primary risks are: (1) **Vercel AI SDK v6 API uncertainty** -- most available examples target v4/v5, and the v6 streaming API may differ significantly; (2) **billing race conditions** in parallel streaming -- one model can succeed while the other fails, creating partial-charge reconciliation problems; and (3) **Reconsider context window overflow** -- sending both model outputs back as context can exceed token limits. All three are mitigable with defensive patterns identified in research, but the SDK uncertainty means Phase 1 must begin with API verification before committing to implementation patterns.

## Key Findings

### Recommended Stack

The stack is pre-selected and appropriate for the use case. The critical integration layer is the **Vercel AI SDK v6** (`ai`, `@ai-sdk/react`, `@ai-sdk/openai`, `@ai-sdk/google`), which provides `streamText()` for server-side LLM streaming and provider-specific adapters. The site deploys as a single Next.js container on Cloud Run with Firebase Auth and Firestore for persistence.

**Core technologies:**
- **Vercel AI SDK v6**: LLM streaming orchestration -- `streamText()` with parallel execution via `Promise.all()`, provider-agnostic interface
- **Next.js 16 Route Handlers**: API layer for SSE streaming endpoints -- server-side key protection, Edge Runtime for unbuffered streaming
- **Firestore**: Conversation history and credit ledger -- real-time listeners, serverless scaling, existing integration with Firebase Auth
- **Stripe**: Credit purchases -- existing integration on parent site, webhook-based fulfillment

**Critical version note:** Vercel AI SDK v6 introduced breaking changes (parts-based messages, changed `streamText` return signature). All implementation must use v6 docs exclusively -- no v4/v5 patterns.

### Expected Features

**Must have (table stakes):**
- Side-by-side streaming display (core value proposition)
- Real-time token-by-token streaming (SSE-based)
- Standard/Expert tier selection (model pair toggle)
- Clean prompt input (textarea with shift+enter, auto-resize)
- Copy output per response
- Graceful error handling (per-model, not generic)
- Session history (in-memory)
- Mobile responsive layout (stacked on mobile, side-by-side on desktop)

**Should have (differentiators):**
- **Reconsider mode** -- THE competitive differentiator; models see peer output and revise. This is what distinguishes this tool from ChatGPT Arena, Poe, and similar products
- Credit/tier system (10/20 credits, already planned)
- Follow-up questions (conversation continuity)
- Persistent prompt history (Firestore-backed)
- Export to Markdown

**Defer (v2+):**
- Blind comparison / arena-style voting (conflicts with "expert assistant" positioning)
- Collaborative sessions (complex, niche)
- Voice input (nice-to-have, not core)
- Diff view between outputs (medium complexity, low urgency)
- Prompt template library (low effort but not critical for launch)

### Architecture Approach

The tool integrates into the existing site's patterns: API routes under `/api/tools/research-assistant/*`, components under `src/components/tools/research-assistant/*`, and shared infrastructure (Firebase Auth, credit system, Cloud Run deployment). The key architectural pattern is a **single multiplexed SSE endpoint** that initiates parallel `streamText()` calls, merges both provider streams via `TransformStream`, and sends named events (`openai-token`, `gemini-token`, `complete`) that the client demultiplexes into side-by-side panels.

**Major components:**
1. **StreamingController** -- Parallel LLM orchestration, stream merging, SSE formatting
2. **CreditManager** -- Provisional deduction before streaming, confirmation/refund on completion (shared with other tools)
3. **ModelClient** -- Provider-agnostic abstraction over OpenAI and Google SDKs
4. **ConversationStore** -- Firestore CRUD for conversation history, context loading for follow-ups
5. **ChatInterface + ResponseDisplay** -- Client-side SSE consumption, real-time rendering, state management
6. **useResearchChat hook** -- Custom React hook encapsulating SSE connection, response state, and streaming lifecycle

### Critical Pitfalls

1. **Stream multiplexing without coordination** -- Two separate EventSource connections cause race conditions, orphaned streams, and no failure correlation. **Avoid by:** Single SSE transport with envelope pattern and coordinated completion events. Must be solved in Phase 1.

2. **Billing race conditions** -- Parallel model calls with one success and one failure create partial-charge problems. **Avoid by:** Pessimistic locking (debit full cost upfront), two-phase commit (PENDING then SUCCESS/FAILED), never refund mid-stream. Must be solved in Phase 1.

3. **Reconsider context window overflow** -- Sending both 2000-token outputs back as context can exceed model limits. **Avoid by:** Pre-flight token counting, per-model budget enforcement, truncation strategy with user warning. Must be solved before enabling Reconsider in Phase 3.

4. **Vercel AI SDK v6 breaking changes** -- v4/v5 examples pervade tutorials and training data. `message.content` is now parts-based, `streamText` return changed. **Avoid by:** Use official v6 docs exclusively, lock to v6.x, audit for v4 patterns in every review.

5. **Cloud Run cold start latency** -- 3-8 seconds of dead air before streaming begins ruins UX. **Avoid by:** `--min-instances=1` for production (~$8/month), "connecting" feedback event sent immediately, optimized container (<150MB).

## Cross-Cutting Concerns

Themes that appeared across multiple research files:

### 1. Vercel AI SDK v6 Uncertainty (STACK, PITFALLS, ARCHITECTURE)
Every research dimension flagged that SDK v6 APIs cannot be fully verified from available sources. The streaming multiplexing pattern, `useChat` multi-instance behavior, parts-based message format, and `streamText` return signature all need verification against current docs before implementation begins. **This is the single highest-risk unknown in the project.**

### 2. Parallel Streaming Coordination (STACK, ARCHITECTURE, PITFALLS)
The core technical challenge -- multiplexing two independent LLM streams -- was identified in all three technical research files. There is strong consensus on the solution: single SSE transport, `TransformStream` merging, named events, coordinated completion. The pattern is well-understood but implementation details depend on SDK v6 specifics.

### 3. Credit System Integrity (STACK, ARCHITECTURE, PITFALLS)
Billing correctness under partial failure conditions was flagged in both ARCHITECTURE (provisional deduction pattern) and PITFALLS (race condition scenarios). STACK provided Firestore transaction patterns. The consensus is two-phase commit with pessimistic locking -- debit upfront, reconcile on completion.

### 4. Reconsider Mode Complexity (FEATURES, PITFALLS, ARCHITECTURE)
All three files that touch Reconsider mode flag it as high-complexity. FEATURES identifies it as the competitive differentiator. PITFALLS warns about context window overflow. ARCHITECTURE shows the data flow requires sequential execution (initial responses must complete before Reconsider can begin). This feature should not be attempted until streaming and billing are rock-solid.

### 5. Integration Over Invention (ARCHITECTURE, STACK)
Both files emphasize that this is a new feature on an existing site, not a greenfield project. Firebase Auth, credit system, Stripe, Cloud Run deployment, and admin dashboard already exist. Reuse is mandatory -- no parallel infrastructure.

## Risk Matrix

| Risk | Severity | Likelihood | Impact | Mitigation | Phase |
|------|----------|------------|--------|------------|-------|
| AI SDK v6 API mismatch | Critical | HIGH | Streaming patterns don't work as coded | Verify docs first, prototype before committing | Phase 1 |
| Billing race conditions | Critical | MEDIUM | Users charged for failed requests | Two-phase commit, pessimistic locking | Phase 1 |
| Context window overflow (Reconsider) | Critical | HIGH | API errors, degraded quality | Token budget enforcement, pre-flight counting | Phase 3 |
| Cloud Run cold starts | High | HIGH | 3-8s dead air, users assume broken | Min instances, instant feedback event | Phase 2 |
| Stream multiplexing bugs | High | MEDIUM | One model failure breaks entire UI | Envelope pattern, per-model error handling | Phase 1 |
| Firestore write amplification | Medium | MEDIUM | Excessive costs at scale | Buffer writes until stream completion | Phase 2 |
| Rate limit cascades | Medium | LOW | Rapid retry burns attempts | Exponential backoff, respect Retry-After | Phase 2 |
| Streaming timeout (infinite hang) | Medium | LOW | Credits debited, no response | AbortSignal timeout, heartbeat events | Phase 2 |
| EventSource vs Fetch API choice | Low | HIGH | EventSource doesn't support POST | Use Fetch + ReadableStream (prototype in Phase 1) | Phase 1 |

## Implications for Roadmap

Based on research, suggested phase structure:

### Phase 1: Foundation and Streaming Proof-of-Concept
**Rationale:** The dependency chain starts here. Credit management, model client abstraction, and basic streaming must work before anything else. This phase also resolves the highest-risk unknown (AI SDK v6 API verification).
**Delivers:** Working parallel streaming endpoint, credit deduction, basic side-by-side UI
**Addresses:** Side-by-side display, streaming responses, tier selection, prompt input, basic error handling
**Avoids:** Stream multiplexing bugs (Critical #1), billing race conditions (Critical #2), SDK v6 breaking changes (Critical #4)
**Key tasks:**
- Verify Vercel AI SDK v6 streaming API against current docs (FIRST task)
- Firestore schema creation (conversations, credits, usage logs)
- CreditManager with two-phase commit pattern
- ModelClient abstraction (OpenAI + Google providers)
- StreamingController with TransformStream merging
- API route `/api/tools/research-assistant/chat`
- Basic ResponseDisplay component (two panels, streaming text)
- Prototype EventSource vs Fetch API for SSE consumption

### Phase 2: Core UX and Production Readiness
**Rationale:** With streaming working, focus shifts to user experience quality and deployment concerns. Cold start mitigation, error UX, mobile layout, and copy functionality make the tool usable.
**Delivers:** Production-quality streaming UX, mobile support, deployment pipeline
**Addresses:** Copy outputs, mobile responsive layout, error handling (per-model messages), session history
**Avoids:** Cold start latency (Critical #5), Firestore write amplification (Moderate #3), CORS issues (Moderate #4), no loading state differentiation (Minor #2)
**Key tasks:**
- Loading state machine (connecting / streaming / finalizing)
- Per-model error messages with graceful degradation
- Mobile responsive layout (stacked vertical)
- Copy button per response
- Session history (in-memory)
- Cloud Run deployment with min-instances=1
- Docker image optimization (<150MB)
- Streaming timeout + heartbeat implementation
- Rate limit handling with exponential backoff

### Phase 3: Conversation Features and Reconsider
**Rationale:** With core streaming solid, add the features that make this product unique. Reconsider mode is the competitive differentiator but requires token budget enforcement and sequential streaming orchestration. Follow-up questions and persistent history share the ConversationStore dependency.
**Delivers:** Reconsider mode, follow-up questions, persistent conversation history
**Addresses:** Reconsider mode (differentiator), follow-up questions, persistent history (Firestore), conversation context management
**Avoids:** Context window overflow (Critical #3), memory bloat from storing full responses in state (Anti-pattern #5)
**Key tasks:**
- ConversationStore (Firestore CRUD, context loading)
- Follow-up question flow (append context, re-stream)
- Token budget enforcement (per-model limits, truncation strategy)
- Reconsider orchestration (sequential: initial responses complete, then peer-context injection)
- ReconsiderDialog UI component
- ConversationHistory sidebar
- Conversation URL routing (`/tools/research-assistant?id=abc123`)

### Phase 4: Polish and Monetization
**Rationale:** With core features working, add export, cost transparency, credit purchase flow integration, and admin analytics. These are lower-risk, well-documented patterns that don't need deep research.
**Delivers:** Export, cost tracking, Stripe integration, admin dashboard
**Addresses:** Export to Markdown, cost tracking (token display), credit purchase flow, admin usage stats
**Key tasks:**
- Markdown export (conversation to .md file)
- Token usage display per response
- Credit balance display with real-time updates
- Stripe checkout integration for credit purchases
- Webhook handler for purchase fulfillment
- Admin dashboard panel (usage stats, tier breakdown)
- Firestore security rules finalization

### Phase Ordering Rationale

- **Phase 1 before Phase 2:** You cannot polish UX for a streaming feature that does not stream yet. The streaming proof-of-concept resolves the highest-risk technical unknowns (SDK v6 API, multiplexing pattern) before investing in UI refinement.
- **Phase 2 before Phase 3:** Production deployment concerns (cold starts, error handling, mobile) must be solved before adding complexity. Reconsider mode on a brittle streaming foundation will compound bugs.
- **Phase 3 before Phase 4:** Reconsider mode is the product's reason to exist. It must ship before polish features. Export, cost tracking, and admin stats are valuable but not differentiating.
- **Billing in Phase 1, not Phase 4:** Credit deduction is coupled to streaming (debit before call, refund on failure). Deferring billing creates a "bolt-on billing" anti-pattern that causes race conditions.
- **Parallelizable within phases:** In Phase 1, Firestore schema + ModelClient + CreditManager can be built simultaneously. In Phase 3, ConversationHistory UI and ReconsiderDialog UI are independent once ConversationStore exists.

### Research Flags

Phases likely needing deeper research during planning:
- **Phase 1:** Vercel AI SDK v6 streaming API verification is MANDATORY before implementation begins. Prototype multiplexed SSE with actual SDK before committing to patterns. Also verify Edge Runtime + Firestore Admin SDK compatibility.
- **Phase 3:** Reconsider mode has no established open-source pattern. Token counting across providers (tiktoken for OpenAI, unknown for Gemini) needs investigation. Sequential streaming orchestration (wait for both, then re-stream with peer context) needs prototyping.

Phases with standard patterns (skip deep research):
- **Phase 2:** Mobile responsive layouts, Docker optimization, Cloud Run deployment, error handling -- all well-documented patterns with abundant examples.
- **Phase 4:** Stripe Checkout, Markdown generation, Firestore security rules, admin dashboards -- standard CRUD and integration patterns.

## Confidence Assessment

| Area | Confidence | Notes |
|------|------------|-------|
| Stack | MEDIUM | Stack is pre-selected (high confidence on choices), but AI SDK v6 API details are unverified (low confidence on implementation patterns) |
| Features | MEDIUM | Table stakes and differentiators well-identified. Competitive landscape based on Jan 2025 training data -- competitors may have evolved |
| Architecture | HIGH | Next.js + Firebase + SSE streaming is an established pattern. Integration with existing site reduces architectural risk. Component boundaries are clean |
| Pitfalls | MEDIUM | Critical pitfalls are well-identified (fundamental distributed systems and API patterns). Specific v6 SDK gotchas need verification against current docs |

**Overall confidence:** MEDIUM -- The architecture is sound and the pitfalls are well-mapped, but the AI SDK v6 uncertainty creates a single-point knowledge gap that must be resolved at the start of Phase 1.

### Gaps to Address

- **Vercel AI SDK v6 streaming API:** Most critical gap. Verify `streamText()` signature, multi-provider concurrent patterns, parts-based message format, and `onFinish` callback shape against official docs before any implementation.
- **EventSource vs Fetch for SSE:** EventSource does not support POST requests. The architecture assumes Fetch + ReadableStream, but this needs prototyping to confirm SSE parsing works correctly client-side.
- **Gemini SDK streaming maturity:** Google AI SDK for Gemini was less mature than OpenAI's as of early 2025. Streaming behavior, error formats, and rate limit headers may differ. Test during Phase 1.
- **Edge Runtime + Firestore Admin:** Firestore Admin SDK may require Node.js runtime (not Edge). If Edge is required for streaming, a different Firestore client approach may be needed. Test during Phase 1.
- **GPT-5.2 and Gemini 3.0 model identifiers:** Model names used in research are assumed. Actual model IDs need verification against provider APIs.
- **Token counting across providers:** OpenAI uses tiktoken, Gemini uses a different tokenizer. Accurate cross-provider token counting for credit billing and Reconsider budget enforcement needs investigation.

## Sources

### Primary (HIGH confidence)
- Next.js Route Handler streaming patterns (official docs, stable since Next.js 13)
- Server-Sent Events specification (MDN, stable web standard)
- Firebase Admin SDK transaction semantics (official Firebase docs)
- Stripe Checkout + webhook integration (official Stripe docs)
- Cloud Run container deployment (GCP official docs)

### Secondary (MEDIUM confidence)
- Vercel AI SDK architecture and `streamText()` patterns (based on v4/v5 docs, v6 inferred)
- Firestore schema design for credit ledgers (common e-commerce pattern)
- LLM comparison tool feature expectations (based on 2023-2024 ecosystem analysis)
- Cloud Run cold start characteristics (evolving, benchmarks from 2024)

### Tertiary (LOW confidence)
- Vercel AI SDK v6 specific API changes (parts-based messages, return types) -- needs doc verification
- GPT-5.2 and Gemini 3.0 model availability and identifiers -- assumed names, not official
- LLM streaming performance benchmarks (TTFT, tokens/sec) -- estimates, not measured
- Gemini SDK streaming API maturity -- limited training data from early 2025

---
*Research completed: 2026-02-13*
*Ready for roadmap: yes*
