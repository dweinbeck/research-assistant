# Domain Pitfalls

**Domain:** Multi-Model LLM Streaming Comparison Tool
**Researched:** 2026-02-13
**Confidence:** MEDIUM (based on framework knowledge + domain patterns, not externally verified)

## Critical Pitfalls

Mistakes that cause rewrites or major issues.

### Pitfall 1: Stream Multiplexing Without Proper Coordination
**What goes wrong:** Sending two independent SSE streams to the browser causes race conditions, out-of-order rendering, and orphaned streams when one model fails. The browser sees two separate `EventSource` connections and has no way to correlate which chunk belongs to which model or detect when one stream ends abnormally.

**Why it happens:** Developers assume they can just call `streamText()` twice and send both responses to the client. Vercel AI SDK v6 returns `Response` objects with SSE bodies, but doesn't provide built-in multiplexing for parallel streams.

**Consequences:**
- One model's failure doesn't stop the other stream (user sees partial results without knowing one failed)
- Browser can't distinguish "model finished" from "connection dropped"
- No way to send metadata (model name, token count) alongside chunks
- Memory leaks from unclosed SSE connections
- Race conditions in UI state updates

**Prevention:**
1. **Single transport channel:** Multiplex both streams server-side into ONE SSE stream
2. **Envelope pattern:** Wrap each chunk with metadata:
   ```typescript
   type StreamEvent = {
     model: 'gpt-4' | 'gemini-pro',
     type: 'chunk' | 'done' | 'error',
     content?: string,
     usage?: TokenUsage,
     error?: string
   }
   ```
3. **Coordinated completion:** Don't end SSE stream until BOTH models finish or error
4. **Use TransformStream:** Merge both AI SDK streams into single response stream

**Detection:** If you have two `useChat` hooks or two `EventSource` instances client-side, you're doing it wrong.

**Phase mapping:** Phase 1 (Core Streaming) must solve this before any UI work.

---

### Pitfall 2: Billing Race Conditions (Debit-Before-Call)
**What goes wrong:** You debit credits, call two models in parallel, one succeeds and one fails. Now you have three problems:
1. Which model's tokens do you bill for?
2. Do you refund the failed model's estimated cost?
3. What if the success happens AFTER you've already started the refund transaction?

**Why it happens:** Developers treat parallel LLM calls like synchronous transactions, ignoring that:
- Models have different latencies (GPT-4 vs Gemini)
- Failures can happen at different stages (connection error vs rate limit vs timeout)
- Firestore transactions can't span multiple async operations

**Consequences:**
- User charged for failed requests
- Double-billing or double-refunds (race condition between success callback and error handler)
- Orphaned debit records when writes fail
- Audit trail breaks (can't trace which billing record maps to which request)

**Prevention:**
1. **Pessimistic locking:** Debit FULL cost for both models upfront, refund unused on completion
2. **Atomic per-model billing:**
   ```typescript
   // Each model gets its own billing record
   const billingRecords = await Promise.all([
     createBillingRecord(userId, 'gpt-4', estimatedCost),
     createBillingRecord(userId, 'gemini-pro', estimatedCost)
   ]);

   // On completion, update the record with actual usage
   await updateBillingRecord(billingRecords[0].id, actualUsage);
   ```
3. **Idempotency keys:** Use request ID + model name as deduplication key
4. **Two-phase commit pattern:**
   - Phase 1: Create PENDING billing record
   - Phase 2: Update to SUCCESS/FAILED with actual tokens
5. **Never refund mid-stream:** Wait until BOTH streams complete before reconciling

**Detection:**
- User reports: "I was charged but got an error"
- Firestore shows billing records without corresponding usage records
- Credits don't match sum of successful requests

**Phase mapping:** Phase 1 must implement billing scaffold. Phase 2 adds refund reconciliation.

---

### Pitfall 3: Reconsider Context Window Overflow
**What goes wrong:** User has two models generate 2000-token responses each. Reconsider sends both outputs back to each model, blowing past context limits (especially for models with 4K-8K windows or if original prompt was already large).

**Why it happens:** Developers don't account for:
- Original prompt + user message + 2 peer outputs can exceed 16K tokens
- Different models have different context limits (GPT-4 Turbo: 128K, Gemini Pro: 32K, older models: 8K)
- Token counting is model-specific (GPT uses tiktoken, Gemini uses different encoding)

**Consequences:**
- Silent truncation (model cuts off peer outputs)
- API errors (400 Bad Request: context too long)
- Degraded Reconsider quality (peer outputs summarized or truncated by API)
- Billing for failed requests

**Prevention:**
1. **Token budget enforcement:**
   ```typescript
   const RECONSIDER_BUDGET = {
     'gpt-4-turbo': 120000, // leave 8K headroom
     'gemini-pro': 28000,
     'claude-3': 180000
   };

   function canReconsider(model, promptTokens, peerOutputTokens) {
     return promptTokens + peerOutputTokens < RECONSIDER_BUDGET[model];
   }
   ```
2. **Truncation strategy:**
   - If peer outputs exceed budget, truncate oldest messages first
   - Or use extractive summarization (show first 500 + last 500 tokens)
   - Never silently truncate — warn user "Peer output too long, showing summary"
3. **Pre-flight token counting:**
   - Use `tiktoken` (GPT) or model-specific tokenizers
   - Count BEFORE calling API
   - Vercel AI SDK v6 doesn't expose token counting — you need separate library
4. **Tier-based limits:**
   - Free tier: disable Reconsider if combined output > 4K tokens
   - Premium tier: allow up to context limit with warning

**Detection:**
- API returns 400 errors on Reconsider requests
- Reconsider responses are noticeably shorter than initial responses
- Usage logs show token counts near context limits

**Phase mapping:** Phase 3 (Reconsider) must implement token budget checks before enabling feature.

---

### Pitfall 4: Vercel AI SDK v6 Breaking Changes Ignored
**What goes wrong:** Code examples from v4/v5 tutorials don't work. `useChat` API changed, message format changed from string to parts-based, streaming responses require different handling.

**Why it happens:**
- Most tutorials/Stack Overflow are for v4 (pre-2024)
- v6 introduced parts-based messages: `message.content` is now `TextPart[] | ImagePart[]`
- `streamText()` return signature changed
- Error handling patterns changed

**Consequences:**
- Type errors: `message.content` is not a string
- Runtime errors: trying to concatenate parts as strings
- Lost functionality: image inputs break
- Streaming doesn't work: expecting v4 streaming API

**Prevention:**
1. **Use official v6 docs exclusively:** https://sdk.vercel.ai/docs (check version selector)
2. **Migration checklist:**
   - `message.content` → `message.content.map(part => part.text).join('')`
   - `onFinish` callback signature changed (now receives full response object)
   - `streamText` returns `{ textStream, fullStream }` not single stream
3. **Type safety:**
   ```typescript
   import { Message } from 'ai';

   function extractText(message: Message): string {
     return message.content
       .filter(part => part.type === 'text')
       .map(part => part.text)
       .join('');
   }
   ```
4. **Don't mix SDK versions:** Lock to v6.x in package.json, audit for v4 patterns

**Detection:**
- TypeScript errors on `message.content`
- Streaming responses render as `[object Object]`
- Image messages break UI

**Phase mapping:** Phase 0 (setup) must use v6 from the start. No v4 code allowed.

---

### Pitfall 5: Cloud Run Cold Start Latency Ruins Streaming UX
**What goes wrong:** User clicks "Compare", waits 3-8 seconds seeing nothing, then both streams start. They assume it's broken and refresh, triggering another cold start.

**Why it happens:**
- Cloud Run scales to zero, cold starts take 2-10s depending on image size
- Next.js on Cloud Run has large container images (300MB+)
- Streaming doesn't start until AFTER cold start completes
- No feedback to user during cold start

**Consequences:**
- Perceived as "broken" or "slow"
- Users refresh, wasting credits on duplicate requests
- Poor first impression
- Increased billing from wasted cold starts

**Prevention:**
1. **Min instances:** Set `--min-instances=1` for production (costs ~$8/month but eliminates cold starts)
2. **Instant feedback:**
   ```typescript
   // Send immediate "connecting" event before LLM calls
   const encoder = new TextEncoder();
   const stream = new TransformStream();
   const writer = stream.writable.getWriter();

   writer.write(encoder.encode(`data: ${JSON.stringify({ type: 'connecting' })}\n\n`));

   // Then start LLM calls
   ```
3. **Optimize container:**
   - Use `.dockerignore` to exclude dev dependencies
   - Multi-stage builds (build stage + runtime stage)
   - Target <150MB image size
4. **Warm-up endpoint:** Use Cloud Scheduler to ping every 5 minutes (keeps instance warm)
5. **Consider Cloud Run gen2:** Faster cold starts than gen1

**Detection:**
- Cloud Run logs show cold start durations >2s
- Analytics show high bounce rate on compare page
- User feedback: "nothing happened"

**Phase mapping:** Phase 2 (production optimization) must address cold starts before launch.

---

## Moderate Pitfalls

### Pitfall 1: Rate Limit Handling Without Retry Backoff
**What goes wrong:** OpenAI/Google APIs return 429 (rate limit). You retry immediately, hit rate limit again, burn through all retry attempts in <1 second. User gets error instead of waiting 2 seconds for quota refresh.

**Prevention:**
- Use exponential backoff: 1s, 2s, 4s, 8s delays
- Respect `Retry-After` headers
- Vercel AI SDK has built-in retry, but verify it's enabled:
  ```typescript
  streamText({
    model: openai('gpt-4'),
    maxRetries: 3, // default is 2
    // SDK handles exponential backoff automatically
  });
  ```
- For free tier users: add rate limit buffer (max 5 requests/minute instead of API's 60/minute)

**Detection:** Logs show multiple 429 errors within same second.

---

### Pitfall 2: No Streaming Timeout (Infinite Hangs)
**What goes wrong:** Model API hangs (rare but happens), stream never completes, user waits forever, credits debited but no response.

**Prevention:**
- Set request timeout: 60s for most models, 120s for o1/reasoning models
- Vercel AI SDK v6:
  ```typescript
  streamText({
    model: openai('gpt-4'),
    abortSignal: AbortSignal.timeout(60000), // 60s
  });
  ```
- Client-side timeout: close EventSource after 90s of no data
- Send heartbeat comments every 15s to detect stalled connections:
  ```typescript
  const heartbeat = setInterval(() => {
    writer.write(encoder.encode(': heartbeat\n\n'));
  }, 15000);
  ```

**Detection:** Users report "stuck loading", billing records without completion timestamps.

---

### Pitfall 3: Firestore Read/Write Amplification
**What goes wrong:** Every stream chunk triggers Firestore write to save conversation. 2000 token response = 2000 Firestore writes = $0.36 per response (Firestore charges $0.18 per 100K writes).

**Prevention:**
- **Buffer writes:** Only save to Firestore when stream completes
- **Use Firestore batched writes:** Combine user message + both model responses into single batch
- **Don't store chunks:** Store final messages only
- **Client-side state:** Use React state for in-progress streams, Firestore for persistence
  ```typescript
  // Bad: write every chunk
  onChunk: (chunk) => firestore.collection('messages').add(chunk)

  // Good: buffer until done
  onFinish: (result) => firestore.collection('messages').add(result)
  ```

**Detection:** Firestore billing shows 100K+ writes for 50 conversations.

---

### Pitfall 4: CORS Errors on Streaming Endpoints
**What goes wrong:** EventSource can't connect to API route, browser console shows CORS errors, streaming fails.

**Prevention:**
- Next.js API routes don't have CORS issues when deployed to same domain
- If using separate backend: set CORS headers on OPTIONS preflight:
  ```typescript
  if (req.method === 'OPTIONS') {
    return new Response(null, {
      headers: {
        'Access-Control-Allow-Origin': '*',
        'Access-Control-Allow-Methods': 'GET, POST, OPTIONS',
      }
    });
  }
  ```
- Vercel deployment: CORS not needed (same origin)
- Cloud Run: add CORS to service config

**Detection:** Browser console shows "CORS policy" errors, EventSource state is CLOSED.

---

### Pitfall 5: No Model-Specific Error Messages
**What goes wrong:** All errors show "Something went wrong", user doesn't know if it's rate limit, invalid API key, or model down.

**Prevention:**
- Parse API error responses:
  ```typescript
  catch (error) {
    if (error.status === 429) return "Rate limited. Please wait 60 seconds.";
    if (error.status === 401) return "API key invalid. Please contact support.";
    if (error.status === 503) return "Model temporarily unavailable. Try again.";
    if (error.code === 'context_length_exceeded') return "Message too long.";
    return "Unexpected error. Please try again.";
  }
  ```
- Show model name in error: "GPT-4 failed: Rate limited"
- Different handling for each model failure in parallel scenario

**Detection:** Support requests asking "what does 'something went wrong' mean?"

---

## Minor Pitfalls

### Pitfall 1: Token Usage Not Displayed
**What goes wrong:** Users don't know how many tokens they used, can't judge if response was "expensive", no transparency in billing.

**Prevention:**
- Vercel AI SDK v6 `onFinish` callback includes usage:
  ```typescript
  streamText({
    onFinish: ({ usage }) => {
      console.log('Prompt tokens:', usage.promptTokens);
      console.log('Completion tokens:', usage.completionTokens);
      // Display to user
    }
  });
  ```
- Show per-model token counts in UI
- Show credit deduction calculation: "GPT-4: 1500 tokens = 3 credits"

---

### Pitfall 2: No Loading State Differentiation
**What goes wrong:** User can't tell if model is "connecting", "thinking", or "streaming". All show same spinner.

**Prevention:**
- Three distinct states:
  1. **Connecting:** Before first chunk (show "Connecting to GPT-4...")
  2. **Streaming:** Chunks arriving (show animated text)
  3. **Finalizing:** Stream done, saving to DB (show "Saving...")
- Use stream events to track state transitions

---

### Pitfall 3: Markdown Rendering During Streaming
**What goes wrong:** Partial markdown renders incorrectly. Code blocks appear as plain text until closing backticks arrive.

**Prevention:**
- Use streaming-aware markdown renderer (e.g., `react-markdown` with incremental parsing)
- Or: buffer code blocks until complete, render prose immediately
- Or: show raw markdown with monospace font until stream completes, then render

---

### Pitfall 4: Browser Back Button Breaks State
**What goes wrong:** User clicks back, returns to comparison page, sees old results mixed with new ones.

**Prevention:**
- Use unique conversation IDs in URL: `/compare?id=abc123`
- Clear state on unmount
- Or: persist to Firestore, restore from conversation ID

---

### Pitfall 5: No Mobile Optimization for Side-by-Side Streams
**What goes wrong:** Two streaming columns on mobile are unreadable (100px wide each).

**Prevention:**
- Mobile: stack vertically (one model above the other)
- Desktop: side-by-side
- Use responsive breakpoints: `@media (min-width: 768px)`

---

## Phase-Specific Warnings

| Phase Topic | Likely Pitfall | Mitigation |
|-------------|---------------|------------|
| Phase 1: Core Streaming | Stream multiplexing (Critical #1) | Research TransformStream patterns BEFORE implementation |
| Phase 1: Billing | Race conditions (Critical #2) | Use two-phase commit, test partial failure scenarios |
| Phase 2: Production Deploy | Cold starts (Critical #5) | Budget for min-instances, implement instant feedback |
| Phase 3: Reconsider | Context overflow (Critical #3) | Implement token counting BEFORE enabling feature |
| Phase 3: Reconsider | Firestore write amplification (Moderate #3) | Buffer all writes until completion |
| Phase 4: Rate Limiting | No retry backoff (Moderate #1) | Verify Vercel AI SDK retry config, add custom backoff if needed |
| All Phases | Vercel AI SDK v6 breaking changes (Critical #4) | Audit for v4 patterns during every code review |

---

## Research Gaps & Confidence Notes

**MEDIUM confidence areas:**
- Vercel AI SDK v6 specifics (knowledge cutoff: Jan 2025, v6 released late 2024)
- Cloud Run gen2 cold start performance (evolving rapidly)
- Firestore write costs (pricing may have changed)

**Would benefit from verification:**
- Vercel AI SDK v6 official docs for streaming multiplexing patterns
- Cloud Run cold start benchmarks for Next.js 16 containers
- Firestore latest pricing for write operations
- OpenAI/Google rate limit policies as of 2026

**HIGH confidence areas:**
- Stream coordination problems (fundamental distributed systems issue)
- Billing race conditions (standard transaction isolation problem)
- Context window limits (well-documented model constraints)
- CORS patterns (stable web standard)

---

## Sources

- Knowledge based on: LLM API documentation patterns (OpenAI, Google), Vercel AI SDK architecture, Cloud Run operational characteristics, Firestore transaction semantics
- Confidence: MEDIUM (no external verification for 2026 specifics, based on Jan 2025 knowledge + domain patterns)
- Recommended verification: Vercel AI SDK v6 official docs, Cloud Run documentation, model provider rate limit pages
