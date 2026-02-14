# Architecture Patterns: Multi-Model LLM Comparison Tool

**Domain:** Research Assistant - LLM Comparison Feature
**Researched:** 2026-02-13
**Overall confidence:** HIGH (based on established Next.js + Firebase patterns)

## Executive Summary

The Research Assistant is a NEW FEATURE being added to an existing Next.js 16 personal brand site, NOT a standalone application. It follows the established architectural patterns used by existing tools (Brand Scraper, AI Assistant, Digital Envelopes):

- **API Routes:** `/api/tools/research-assistant/*` following existing convention
- **Components:** `src/components/tools/research-assistant/*` and `src/components/admin/research-assistant/*`
- **Database:** Firestore collections mirroring existing credit ledger patterns
- **Auth:** Existing Firebase Auth (Google Sign-In)
- **Billing:** Existing Stripe + Firestore credit system
- **Deployment:** Existing GCP Cloud Run container (no new infrastructure)

The key architectural challenge is **parallel streaming from multiple LLM providers** (OpenAI + Google) with real-time token-by-token updates to the UI, while maintaining conversation history and credit accounting.

## Recommended Architecture

### High-Level Data Flow

```
User Browser
    |
    | (1) POST /api/tools/research-assistant/chat
    v
Next.js API Route (Edge Runtime)
    |
    | (2) Validate credit balance (Firestore)
    | (3) Deduct provisional credits
    v
Parallel LLM Streaming Controller
    |
    +----> (4a) OpenAI SDK (streaming)
    |
    +----> (4b) Google AI SDK (streaming)
    |
    v (5) Merge streams + format SSE
    |
Next.js Response (Server-Sent Events)
    |
    v (6) Real-time UI updates (EventSource)
    |
User Browser (side-by-side display)
    |
    v (7) On completion: save to Firestore
    |
Firestore Collections:
  - research_conversations (history)
  - user_credits (ledger)
  - research_usage_logs (analytics)
```

### Component Boundaries

| Component | Responsibility | Communicates With | Location |
|-----------|---------------|-------------------|----------|
| **ChatInterface** | User input, tier selection, display streaming responses | ChatAPI, TierSelector, ResponseDisplay | `src/components/tools/research-assistant/ChatInterface.tsx` |
| **ResponseDisplay** | Side-by-side streaming panels, syntax highlighting, copy buttons | SSEClient | `src/components/tools/research-assistant/ResponseDisplay.tsx` |
| **ConversationHistory** | Display past conversations, load for follow-ups | Firestore via API | `src/components/tools/research-assistant/ConversationHistory.tsx` |
| **TierSelector** | Standard vs Expert tier choice, credit cost display | ChatInterface | `src/components/tools/research-assistant/TierSelector.tsx` |
| **ChatAPI** | POST /chat, handle SSE connection, parse events | StreamingController | `/api/tools/research-assistant/chat.ts` |
| **StreamingController** | Parallel LLM calls, merge streams, format SSE | OpenAI SDK, Google AI SDK | `/api/tools/research-assistant/lib/streamingController.ts` |
| **CreditManager** | Check balance, deduct credits, refund on error | Firestore `user_credits` | `/api/tools/research-assistant/lib/creditManager.ts` (shared with other tools) |
| **ConversationStore** | Save/load conversations, append messages | Firestore `research_conversations` | `/api/tools/research-assistant/lib/conversationStore.ts` |
| **ModelClient** | Abstraction for OpenAI/Gemini API calls | Provider SDKs | `/api/tools/research-assistant/lib/modelClient.ts` |

### Data Flow Details

#### Flow 1: New Query (Standard Tier)

1. **User types prompt** → ChatInterface component
2. **Select tier** → TierSelector (Standard = 2 credits)
3. **Submit** → POST `/api/tools/research-assistant/chat`
   ```typescript
   {
     prompt: "Explain quantum computing",
     tier: "standard",
     conversationId?: string // optional for follow-ups
   }
   ```
4. **API Route validates:**
   - User authenticated (Firebase Auth)
   - Credit balance >= 2 (CreditManager)
   - Prompt not empty, < 4000 chars
5. **Deduct provisional credits** (prevent double-spend during streaming)
6. **StreamingController initiates parallel calls:**
   - OpenAI: `gpt-5.2-instant` with `stream: true`
   - Gemini: `gemini-2.5-flash` with streaming
7. **Merge streams** into single SSE response:
   ```
   event: openai-token
   data: {"token": "Quantum"}

   event: gemini-token
   data: {"token": "Quantum"}

   event: openai-done
   data: {"tokens": 150, "model": "gpt-5.2-instant"}

   event: gemini-done
   data: {"tokens": 145, "model": "gemini-2.5-flash"}

   event: complete
   data: {"conversationId": "conv_xyz", "creditsUsed": 2}
   ```
8. **Client (ResponseDisplay) renders tokens** in real-time
9. **On completion:**
   - Save conversation to Firestore
   - Log usage to `research_usage_logs`
   - Return conversation ID for follow-ups

#### Flow 2: Follow-Up Query

Same as Flow 1, but includes `conversationId` in request. API loads previous messages and appends to context before calling LLMs.

#### Flow 3: Reconsider (re-run with new context)

Client sends original prompt + selected winner + new instructions:
```typescript
{
  prompt: "Explain quantum computing",
  tier: "expert",
  reconsiderContext: {
    previousWinner: "openai",
    previousResponse: "...",
    instructions: "Focus more on practical applications"
  }
}
```

API constructs new prompt incorporating context, charges credits again, streams new responses.

### API Route Structure

```
/api/tools/research-assistant/
├── chat.ts                    # Main streaming endpoint (POST)
├── conversations/
│   ├── [id].ts               # GET conversation by ID
│   └── list.ts               # GET user's conversation list
├── lib/
│   ├── streamingController.ts # Parallel LLM orchestration
│   ├── modelClient.ts         # OpenAI + Gemini SDK wrappers
│   ├── creditManager.ts       # Credit check/deduct/refund
│   ├── conversationStore.ts   # Firestore CRUD
│   ├── types.ts               # Shared TypeScript types
│   └── config.ts              # Tier definitions, model mappings
└── admin/
    └── usage-stats.ts         # Admin analytics (credit usage by tier)
```

### File Structure (Full Integration)

```
src/
├── app/
│   └── tools/
│       └── research-assistant/
│           ├── page.tsx                    # Main tool page
│           └── layout.tsx                  # Tool-specific layout
├── components/
│   ├── tools/
│   │   └── research-assistant/
│   │       ├── ChatInterface.tsx           # Main UI container
│   │       ├── ResponseDisplay.tsx         # Side-by-side streaming panels
│   │       ├── ConversationHistory.tsx     # Past conversations sidebar
│   │       ├── TierSelector.tsx            # Standard vs Expert choice
│   │       ├── FollowUpInput.tsx           # Context-aware input
│   │       ├── ReconsiderDialog.tsx        # Reconsider flow UI
│   │       └── ExportButton.tsx            # Export conversation
│   └── admin/
│       └── research-assistant/
│           └── UsageStats.tsx              # Admin dashboard panel
└── lib/
    └── hooks/
        └── useResearchChat.ts              # Client hook for SSE

api/
└── tools/
    └── research-assistant/
        ├── chat.ts                         # Main streaming endpoint
        ├── conversations/
        │   ├── [id].ts
        │   └── list.ts
        ├── lib/
        │   ├── streamingController.ts
        │   ├── modelClient.ts
        │   ├── creditManager.ts
        │   ├── conversationStore.ts
        │   ├── types.ts
        │   └── config.ts
        └── admin/
            └── usage-stats.ts
```

### Firestore Schema

```typescript
// Collection: research_conversations
{
  id: string,                    // Auto-generated
  userId: string,                // Firebase Auth UID
  createdAt: Timestamp,
  updatedAt: Timestamp,
  tier: "standard" | "expert",
  messages: [
    {
      role: "user" | "assistant",
      content: string,
      provider?: "openai" | "gemini",  // for assistant messages
      tokens?: number,
      timestamp: Timestamp
    }
  ],
  totalCreditsUsed: number,
  metadata: {
    reconsiderCount: number,
    averageResponseTime: number
  }
}

// Collection: user_credits (existing, shared)
{
  userId: string,
  balance: number,
  transactions: [
    {
      type: "deduct" | "refund" | "purchase",
      amount: number,
      tool: "research-assistant" | "brand-scraper" | ...,
      timestamp: Timestamp,
      metadata: { conversationId?: string, tier?: string }
    }
  ]
}

// Collection: research_usage_logs
{
  id: string,
  userId: string,
  conversationId: string,
  timestamp: Timestamp,
  tier: "standard" | "expert",
  creditsCharged: number,
  models: {
    openai: { model: string, tokens: number, latency: number },
    gemini: { model: string, tokens: number, latency: number }
  }
}
```

## Patterns to Follow

### Pattern 1: Edge Runtime for Streaming

**What:** Use Next.js Edge Runtime for `/api/tools/research-assistant/chat.ts` to enable streaming responses without buffering.

**When:** Any API route that needs to stream LLM responses in real-time.

**Example:**
```typescript
// api/tools/research-assistant/chat.ts
export const runtime = 'edge';

export async function POST(req: Request) {
  const { prompt, tier, conversationId } = await req.json();

  // Validate and deduct credits (use Firestore Admin SDK)
  const userId = await getUserIdFromAuth(req);
  await creditManager.deduct(userId, TIER_COSTS[tier]);

  // Create streaming controller
  const controller = new StreamingController(tier);
  const stream = await controller.streamComparison(prompt, conversationId);

  // Return SSE response
  return new Response(stream, {
    headers: {
      'Content-Type': 'text/event-stream',
      'Cache-Control': 'no-cache',
      'Connection': 'keep-alive',
    },
  });
}
```

### Pattern 2: Parallel Stream Merging

**What:** Initiate both LLM calls simultaneously and interleave their tokens into a single SSE stream.

**When:** Comparing multiple models side-by-side with real-time updates.

**Example:**
```typescript
// lib/streamingController.ts
export class StreamingController {
  async streamComparison(prompt: string, conversationId?: string) {
    const openaiStream = this.modelClient.streamOpenAI(prompt, conversationId);
    const geminiStream = this.modelClient.streamGemini(prompt, conversationId);

    const { readable, writable } = new TransformStream();
    const writer = writable.getWriter();

    // Merge streams
    Promise.all([
      this.pipeStream(openaiStream, writer, 'openai'),
      this.pipeStream(geminiStream, writer, 'gemini'),
    ]).then(() => {
      writer.write(this.formatSSE('complete', { conversationId }));
      writer.close();
    });

    return readable;
  }

  private async pipeStream(
    stream: AsyncIterable<string>,
    writer: WritableStreamDefaultWriter,
    provider: 'openai' | 'gemini'
  ) {
    for await (const token of stream) {
      await writer.write(this.formatSSE(`${provider}-token`, { token }));
    }
    await writer.write(this.formatSSE(`${provider}-done`, { provider }));
  }

  private formatSSE(event: string, data: any): Uint8Array {
    return new TextEncoder().encode(`event: ${event}\ndata: ${JSON.stringify(data)}\n\n`);
  }
}
```

### Pattern 3: Client-Side SSE Handling

**What:** Use EventSource API (or polyfill) to consume server-sent events and update UI.

**When:** Displaying real-time streaming responses in React components.

**Example:**
```typescript
// lib/hooks/useResearchChat.ts
export function useResearchChat() {
  const [responses, setResponses] = useState({ openai: '', gemini: '' });
  const [isStreaming, setIsStreaming] = useState(false);

  const sendMessage = async (prompt: string, tier: string) => {
    setIsStreaming(true);
    setResponses({ openai: '', gemini: '' });

    const eventSource = new EventSource(
      `/api/tools/research-assistant/chat?prompt=${encodeURIComponent(prompt)}&tier=${tier}`
    );

    eventSource.addEventListener('openai-token', (e) => {
      const { token } = JSON.parse(e.data);
      setResponses((prev) => ({ ...prev, openai: prev.openai + token }));
    });

    eventSource.addEventListener('gemini-token', (e) => {
      const { token } = JSON.parse(e.data);
      setResponses((prev) => ({ ...prev, gemini: prev.gemini + token }));
    });

    eventSource.addEventListener('complete', () => {
      setIsStreaming(false);
      eventSource.close();
    });

    eventSource.onerror = () => {
      setIsStreaming(false);
      eventSource.close();
    };
  };

  return { responses, isStreaming, sendMessage };
}
```

### Pattern 4: Provisional Credit Deduction

**What:** Deduct credits BEFORE streaming starts to prevent race conditions. Refund if stream fails.

**When:** Any credit-based streaming operation.

**Example:**
```typescript
// lib/creditManager.ts
export class CreditManager {
  async deductProvisional(userId: string, amount: number): Promise<string> {
    const txId = `tx_${Date.now()}`;
    const userRef = db.collection('user_credits').doc(userId);

    await db.runTransaction(async (transaction) => {
      const doc = await transaction.get(userRef);
      const currentBalance = doc.data()?.balance || 0;

      if (currentBalance < amount) {
        throw new Error('Insufficient credits');
      }

      transaction.update(userRef, {
        balance: currentBalance - amount,
        transactions: FieldValue.arrayUnion({
          id: txId,
          type: 'deduct',
          amount,
          tool: 'research-assistant',
          timestamp: FieldValue.serverTimestamp(),
          status: 'provisional'
        })
      });
    });

    return txId;
  }

  async confirmTransaction(userId: string, txId: string) {
    // Mark transaction as confirmed in Firestore
  }

  async refundTransaction(userId: string, txId: string, amount: number) {
    // Refund credits if stream failed
  }
}
```

### Pattern 5: Conversation Context Management

**What:** Load previous messages from Firestore, append to LLM context for follow-ups.

**When:** User continues a conversation thread.

**Example:**
```typescript
// lib/conversationStore.ts
export class ConversationStore {
  async loadContext(conversationId: string): Promise<Message[]> {
    const doc = await db.collection('research_conversations').doc(conversationId).get();
    return doc.data()?.messages || [];
  }

  async appendMessage(conversationId: string, message: Message) {
    await db.collection('research_conversations').doc(conversationId).update({
      messages: FieldValue.arrayUnion(message),
      updatedAt: FieldValue.serverTimestamp()
    });
  }

  formatContextForLLM(messages: Message[]): string {
    return messages
      .map(m => `${m.role === 'user' ? 'User' : 'Assistant'}: ${m.content}`)
      .join('\n\n');
  }
}
```

## Anti-Patterns to Avoid

### Anti-Pattern 1: Buffered Streaming

**What:** Collecting entire LLM response before sending to client.

**Why bad:** Defeats the purpose of streaming UX; increases latency; wastes memory.

**Instead:** Use Edge Runtime + TransformStream to pipe tokens as they arrive.

### Anti-Pattern 2: Sequential LLM Calls

**What:** Waiting for OpenAI to finish before calling Gemini.

**Why bad:** Doubles total latency; poor user experience.

**Instead:** Call both models in parallel using `Promise.all()` or concurrent async iterators.

### Anti-Pattern 3: Client-Side LLM API Keys

**What:** Exposing API keys to browser to make direct LLM calls.

**Why bad:** Security risk; no credit control; exposes usage patterns.

**Instead:** Proxy all LLM calls through Next.js API routes with server-side keys.

### Anti-Pattern 4: No Error Handling for Partial Streams

**What:** Assuming both streams will always complete successfully.

**Why bad:** Network issues, rate limits, or API errors will break UI without recovery.

**Instead:** Wrap each stream in try-catch, send error events via SSE, allow partial results.

**Example:**
```typescript
private async pipeStream(stream, writer, provider) {
  try {
    for await (const token of stream) {
      await writer.write(this.formatSSE(`${provider}-token`, { token }));
    }
    await writer.write(this.formatSSE(`${provider}-done`, { provider }));
  } catch (error) {
    await writer.write(this.formatSSE(`${provider}-error`, {
      message: error.message,
      fallback: 'This model encountered an error. The other model may still complete.'
    }));
  }
}
```

### Anti-Pattern 5: Storing Full Responses in User State

**What:** Keeping entire conversation history in React state indefinitely.

**Why bad:** Memory bloat; state updates on every token slow down rendering.

**Instead:** Store only current streaming responses in state. Load history from Firestore on demand.

## Scalability Considerations

| Concern | At 100 users | At 10K users | At 1M users |
|---------|--------------|--------------|-------------|
| **Concurrent streams** | Direct API calls (no queue) | Add rate limiting per user | Introduce Redis queue + worker pool |
| **Firestore writes** | Real-time per message | Batch writes (every 10 messages) | Sharded collections by userId prefix |
| **Credit ledger** | Single document per user | Firestore transactions OK | Move to Cloud SQL for ACID guarantees |
| **SSE connections** | Cloud Run (512 MB instances) | Cloud Run (2 GB instances, autoscale) | Separate WebSocket service (GKE) |
| **Cost (LLM APIs)** | ~$50/month | ~$5K/month | Negotiate enterprise pricing with OpenAI/Google |

### Build Order Implications

**Phase 1 dependencies (Foundation):**
1. Firestore schema + seed data (no dependencies)
2. Credit manager (depends on schema)
3. Model client abstraction (no dependencies)
4. API route skeleton (depends on schema, credit manager)

**Phase 2 dependencies (Streaming):**
1. Streaming controller (depends on model client)
2. API route integration (depends on streaming controller, credit manager)
3. Basic UI (depends on working API)

**Phase 3 dependencies (Features):**
1. Conversation store (depends on Firestore schema)
2. Follow-up logic (depends on conversation store, streaming controller)
3. Reconsider flow (depends on follow-up logic)
4. History UI (depends on conversation store)

**Critical path:** Credit manager → Streaming controller → API route → UI

**Parallelizable:**
- Firestore schema design + UI mockups (can happen simultaneously)
- Model client abstraction + Credit manager (independent)
- History UI + Reconsider dialog (both need conversation store, but independent of each other)

## Integration Points with Existing System

### 1. Firebase Auth

**Existing pattern:** All tools use Firebase Auth for user identification.

**Integration:** Reuse existing `getAuth()` middleware in API routes.

**Code location:** Likely in `src/lib/auth.ts` or similar.

**No new code needed** — just import existing helper.

### 2. Credit System

**Existing pattern:** Stripe purchases add credits to `user_credits` Firestore collection. Tools deduct credits per action.

**Integration:** Use same `CreditManager` class (or create if doesn't exist) with tool-specific transaction metadata.

**New addition:** `tool: "research-assistant"` in transaction objects.

### 3. Tool Navigation

**Existing pattern:** Tools appear in dashboard at `/tools` with cards linking to `/tools/[tool-name]`.

**Integration:** Add Research Assistant card to dashboard.

**Code location:** Likely `src/app/tools/page.tsx` or `src/components/ToolsGrid.tsx`.

**New addition:** One card component with icon, title, description, credit cost.

### 4. Admin Analytics

**Existing pattern:** Admin dashboard shows usage stats per tool.

**Integration:** Add Research Assistant section to admin dashboard.

**Code location:** `src/app/admin/page.tsx` or `src/components/admin/UsageStats.tsx`.

**New addition:** Query `research_usage_logs` for total queries, credits spent, tier breakdown.

### 5. Deployment

**Existing pattern:** Single Next.js app deployed to Cloud Run, no per-tool infrastructure.

**Integration:** No changes needed. New API routes and pages automatically included in build.

**Terraform:** No updates required (path-based routing already configured).

## Confidence Assessment

| Area | Confidence | Reasoning |
|------|------------|-----------|
| **Overall architecture** | HIGH | Next.js streaming + Firebase is well-established pattern |
| **Parallel streaming** | HIGH | Standard approach using TransformStream + SSE |
| **Integration patterns** | HIGH | Following existing tool structure reduces risk |
| **Firestore schema** | MEDIUM | Schema design sound, but scale unknowns (10K+ conversations) |
| **Credit provisioning** | MEDIUM | Transaction approach solid, but race condition edge cases need testing |
| **Model client abstractions** | MEDIUM | OpenAI SDK well-documented, Gemini SDK less mature (as of early 2025) |

## Sources

**HIGH confidence (established patterns):**
- Next.js Edge Runtime streaming: Official Next.js 13+ docs on Route Handlers with streaming
- Server-Sent Events (SSE): MDN EventSource API documentation
- Firebase Admin SDK transactions: Firebase official documentation

**MEDIUM confidence (inferred from context):**
- Existing tool structure: Based on provided file paths (`/api/tools/brand-scraper/*`)
- Credit system patterns: Common e-commerce pattern (Stripe + Firestore ledger)

**LOW confidence (assumptions):**
- Gemini SDK streaming API: Training data from early 2025, API may have evolved
- Exact Cloud Run configuration: Not verified from actual Terraform files

## Open Questions for Phase-Specific Research

1. **Gemini SDK streaming syntax:** Verify exact API for `gemini-2.5-flash` streaming (likely changed since training cutoff). Check official Google AI SDK docs during Phase 1.

2. **Edge Runtime + Firestore Admin:** Confirm Firestore Admin SDK works in Edge Runtime (may need Node.js runtime for full SDK). Test during Phase 1 prototyping.

3. **EventSource vs Fetch streaming:** EventSource doesn't support POST requests. May need to use Fetch API with ReadableStream instead. Prototype during Phase 2.

4. **Rate limiting:** How to handle OpenAI/Gemini rate limits gracefully? Queue system needed? Research during Phase 3 (optimization).

5. **Token counting accuracy:** How to accurately count tokens for credit billing when streaming? Investigate provider APIs during Phase 1.

## Next Steps for Roadmap Creator

**Use this architecture to:**

1. **Define Phase 1 (Foundation):** Start with non-streaming basics (schema, credit manager, simple API route) to prove integration.

2. **Define Phase 2 (Core Streaming):** Tackle parallel streaming after foundation is solid.

3. **Define Phase 3 (Features):** Add conversation history, follow-ups, reconsider after core works.

4. **Flag research needs:**
   - Phase 1: Deep dive on Gemini SDK streaming syntax
   - Phase 1: Verify Edge Runtime + Firestore Admin compatibility
   - Phase 2: EventSource vs Fetch API decision (requires prototyping)

5. **Highlight critical path:** Credit manager → Streaming controller → API route must be sequential. UI can proceed in parallel once API contract is defined.

6. **Note parallelizable work:** Schema design, UI mockups, model client abstraction, credit manager can all happen simultaneously in Phase 1.
