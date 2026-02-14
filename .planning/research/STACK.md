# Technology Stack

**Project:** Multi-Model LLM Streaming Comparison Tool
**Researched:** 2026-02-13
**Overall Confidence:** MEDIUM (training data only, tools unavailable for verification)

## Stack Overview

This stack is **pre-determined** per project requirements. Research focuses on HOW to use these tools for multi-model parallel streaming, not what tools to choose.

## Core Framework

| Technology | Version | Purpose | Why |
|------------|---------|---------|-----|
| Next.js | 16.x | App framework, API routes | Pre-selected. Supports React Server Components, streaming responses, and route handlers for SSE endpoints |
| React | 19.x | UI framework | Pre-selected with Next.js 16. Concurrent rendering features support parallel streaming UI updates |
| Tailwind CSS | 4.x | Styling | Pre-selected. Utility-first CSS for rapid UI development of side-by-side comparison layout |
| Biome | v2.3 | Linting/formatting | Pre-selected. Fast Rust-based toolchain for code quality |

**Confidence:** HIGH (versions specified in requirements)

## LLM Integration

| Technology | Version | Purpose | Why |
|------------|---------|---------|-----|
| Vercel AI SDK (ai) | v6+ (latest) | Core streaming orchestration | Industry-standard abstraction for LLM streaming. `streamText()` API handles provider multiplexing, token streaming, and error handling |
| @ai-sdk/react | v6+ | React hooks for streaming | Provides `useChat()`, `useCompletion()` hooks for client-side stream consumption. Manages stream state, loading, errors |
| @ai-sdk/openai | Latest | OpenAI provider (GPT-5.2) | Official provider adapter. Handles GPT-5.2 Instant and Thinking models with streaming support |
| @ai-sdk/google | Latest | Google provider (Gemini) | Official provider adapter. Handles Gemini 2.5 Flash and 3.0 Pro with streaming support |

**Confidence:** MEDIUM (cannot verify v6+ specific APIs without access to current docs)

### Multi-Model Streaming Architecture

**Pattern: Parallel streamText() with Multiplexed SSE**

```typescript
// Server: Next.js Route Handler
// /app/api/compare/route.ts

import { streamText } from 'ai';
import { openai } from '@ai-sdk/openai';
import { google } from '@ai-sdk/google';

export async function POST(req: Request) {
  const { prompt, tier } = await req.json();

  // Define model pairs by tier
  const models = tier === 'expert'
    ? [
        { provider: google('gemini-3.0-pro'), name: 'gemini' },
        { provider: openai('gpt-5.2-thinking'), name: 'openai' }
      ]
    : [
        { provider: google('gemini-2.5-flash'), name: 'gemini' },
        { provider: openai('gpt-5.2-instant'), name: 'openai' }
      ];

  // Parallel streamText() calls
  const streams = models.map(async ({ provider, name }) => {
    const result = await streamText({
      model: provider,
      prompt,
      temperature: 0.7,
      maxTokens: 2048,
    });

    return { name, stream: result.textStream };
  });

  const resolvedStreams = await Promise.all(streams);

  // Multiplex streams into single SSE response
  const encoder = new TextEncoder();
  const stream = new ReadableStream({
    async start(controller) {
      // Consume both streams in parallel
      await Promise.all(
        resolvedStreams.map(async ({ name, stream }) => {
          for await (const chunk of stream) {
            const sseMessage = `event: ${name}\ndata: ${JSON.stringify({ text: chunk })}\n\n`;
            controller.enqueue(encoder.encode(sseMessage));
          }
          // Send completion event
          controller.enqueue(encoder.encode(`event: ${name}-done\ndata: {}\n\n`));
        })
      );
      controller.close();
    },
  });

  return new Response(stream, {
    headers: {
      'Content-Type': 'text/event-stream',
      'Cache-Control': 'no-cache',
      'Connection': 'keep-alive',
    },
  });
}
```

**Rationale:**
- `Promise.all()` on `streamText()` calls enables true parallel execution
- Named SSE events (`event: gemini`, `event: openai`) allow client to demultiplex
- Each stream writes to shared controller asynchronously
- `-done` events signal stream completion for UI state management

**Confidence:** MEDIUM (pattern based on v4/v5 API structure, v6+ may differ)

**Client: Side-by-Side Rendering**

```typescript
// /app/compare/page.tsx
'use client';

import { useState, useEffect } from 'react';

export default function ComparePage() {
  const [geminiText, setGeminiText] = useState('');
  const [openaiText, setOpenaiText] = useState('');
  const [isLoading, setIsLoading] = useState(false);

  const handleSubmit = async (prompt: string, tier: string) => {
    setIsLoading(true);
    setGeminiText('');
    setOpenaiText('');

    const response = await fetch('/api/compare', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ prompt, tier }),
    });

    const reader = response.body!.getReader();
    const decoder = new TextDecoder();

    while (true) {
      const { done, value } = await reader.read();
      if (done) break;

      const chunk = decoder.decode(value);
      const lines = chunk.split('\n');

      for (const line of lines) {
        if (line.startsWith('event: gemini\n')) {
          const dataLine = lines[lines.indexOf(line) + 1];
          if (dataLine?.startsWith('data: ')) {
            const { text } = JSON.parse(dataLine.slice(6));
            setGeminiText(prev => prev + text);
          }
        } else if (line.startsWith('event: openai\n')) {
          const dataLine = lines[lines.indexOf(line) + 1];
          if (dataLine?.startsWith('data: ')) {
            const { text } = JSON.parse(dataLine.slice(6));
            setOpenaiText(prev => prev + text);
          }
        }
      }
    }

    setIsLoading(false);
  };

  return (
    <div className="grid grid-cols-2 gap-4">
      <div>
        <h2>Gemini</h2>
        <div className="whitespace-pre-wrap">{geminiText}</div>
      </div>
      <div>
        <h2>OpenAI</h2>
        <div className="whitespace-pre-wrap">{openaiText}</div>
      </div>
    </div>
  );
}
```

**Alternative: @ai-sdk/react hooks (if multi-stream supported in v6+)**

```typescript
// MAY be available in v6+ - requires verification
import { useChat } from '@ai-sdk/react';

const { messages: geminiMessages, append: appendGemini } = useChat({
  api: '/api/compare/gemini',
});

const { messages: openaiMessages, append: appendOpenai } = useChat({
  api: '/api/compare/openai',
});

// Trigger both in parallel
await Promise.all([
  appendGemini({ content: prompt }),
  appendOpenai({ content: prompt }),
]);
```

**Confidence:** LOW (useChat() multi-instance pattern unverified for v6+)

## Backend Services

| Technology | Version | Purpose | Why |
|------------|---------|---------|-----|
| Firebase Auth | Latest | User authentication | Pre-selected. Handles sign-in, JWT validation, integrates with Firestore security rules |
| Firestore | Latest | Prompt history, credit tracking | Pre-selected. Real-time document store for user data, prompt logs, credit balance |
| Cloud Run | Latest | Docker deployment | Pre-selected. Serverless container deployment on GCP. Auto-scales, integrates with Firebase |

**Confidence:** HIGH (standard Firebase + GCP integration)

### Firestore Schema

```typescript
// /firestore-schema.ts
interface User {
  uid: string;
  email: string;
  credits: number;
  createdAt: Timestamp;
  subscription?: 'free' | 'standard' | 'expert';
}

interface Prompt {
  id: string;
  userId: string;
  prompt: string;
  tier: 'standard' | 'expert';
  responses: {
    gemini: {
      text: string;
      tokensUsed: number;
      completedAt: Timestamp;
    };
    openai: {
      text: string;
      tokensUsed: number;
      completedAt: Timestamp;
    };
  };
  reconsiderStep?: {
    gemini: string;
    openai: string;
  };
  creditsCharged: number;
  createdAt: Timestamp;
}

interface CreditTransaction {
  id: string;
  userId: string;
  amount: number; // negative for usage, positive for purchase
  type: 'prompt' | 'purchase' | 'bonus';
  promptId?: string;
  stripePaymentIntentId?: string;
  createdAt: Timestamp;
}
```

**Rationale:**
- Denormalized `responses` for fast read access (comparison view)
- `creditsCharged` snapshot prevents retroactive pricing disputes
- `CreditTransaction` log for audit trail and billing reconciliation

**Confidence:** HIGH (standard Firestore patterns)

## Billing

| Technology | Version | Purpose | Why |
|------------|---------|---------|-----|
| Stripe | Latest | Payment processing | Pre-selected existing system. Credit purchase flow already implemented in parent site |
| @stripe/stripe-js | Latest | Client-side checkout | Official client library for Stripe Checkout/Payment Intents |

**Integration Pattern:**

```typescript
// /app/api/credits/purchase/route.ts
import Stripe from 'stripe';

const stripe = new Stripe(process.env.STRIPE_SECRET_KEY!);

export async function POST(req: Request) {
  const { userId, creditAmount } = await req.json();

  const session = await stripe.checkout.sessions.create({
    payment_method_types: ['card'],
    line_items: [{
      price_data: {
        currency: 'usd',
        product_data: { name: `${creditAmount} Research Credits` },
        unit_amount: creditAmount * 10, // $0.10 per credit
      },
      quantity: 1,
    }],
    mode: 'payment',
    success_url: `${process.env.NEXT_PUBLIC_URL}/credits/success`,
    cancel_url: `${process.env.NEXT_PUBLIC_URL}/credits/cancel`,
    metadata: { userId, creditAmount },
  });

  return Response.json({ sessionId: session.id });
}

// Webhook handler
// /app/api/webhooks/stripe/route.ts
export async function POST(req: Request) {
  const sig = req.headers.get('stripe-signature')!;
  const event = stripe.webhooks.constructEvent(
    await req.text(),
    sig,
    process.env.STRIPE_WEBHOOK_SECRET!
  );

  if (event.type === 'checkout.session.completed') {
    const session = event.data.object;
    const { userId, creditAmount } = session.metadata;

    // Update Firestore
    await firestore.collection('users').doc(userId).update({
      credits: FieldValue.increment(Number(creditAmount)),
    });

    // Log transaction
    await firestore.collection('creditTransactions').add({
      userId,
      amount: Number(creditAmount),
      type: 'purchase',
      stripePaymentIntentId: session.payment_intent,
      createdAt: FieldValue.serverTimestamp(),
    });
  }

  return new Response(null, { status: 200 });
}
```

**Confidence:** HIGH (standard Stripe integration pattern)

## Testing

| Technology | Version | Purpose | Why |
|------------|---------|---------|-----|
| Vitest | Latest | Test runner | Pre-selected. Fast, Vite-powered test runner with ESM support |
| @testing-library/react | Latest | Component testing | Industry standard for React component testing |
| @testing-library/user-event | Latest | User interaction simulation | Simulates real user interactions for integration tests |
| msw (Mock Service Worker) | Latest | API mocking | Intercepts network requests for testing streaming endpoints |

**Testing Strategy:**

```typescript
// Example: Test parallel streaming
import { describe, it, expect, vi } from 'vitest';
import { http, HttpResponse } from 'msw';
import { setupServer } from 'msw/node';

const server = setupServer(
  http.post('/api/compare', async ({ request }) => {
    const encoder = new TextEncoder();
    const stream = new ReadableStream({
      start(controller) {
        // Mock gemini stream
        controller.enqueue(encoder.encode('event: gemini\ndata: {"text":"Hello"}\n\n'));
        controller.enqueue(encoder.encode('event: gemini-done\ndata: {}\n\n'));

        // Mock openai stream
        controller.enqueue(encoder.encode('event: openai\ndata: {"text":"Hi"}\n\n'));
        controller.enqueue(encoder.encode('event: openai-done\ndata: {}\n\n'));

        controller.close();
      },
    });

    return new HttpResponse(stream, {
      headers: { 'Content-Type': 'text/event-stream' },
    });
  })
);

describe('Multi-model streaming', () => {
  it('renders both streams side-by-side', async () => {
    // Test implementation
  });
});
```

**Confidence:** MEDIUM (MSW pattern for SSE mocking requires verification)

## Development Dependencies

| Package | Version | Purpose |
|---------|---------|---------|
| TypeScript | Latest 5.x | Type safety |
| @types/node | Latest | Node.js types |
| @types/react | Latest | React types |
| tsx | Latest | TypeScript execution for scripts |
| firebase-admin | Latest | Server-side Firebase SDK for API routes |

## Environment Variables

```bash
# .env.example
# LLM Providers
OPENAI_API_KEY=sk-...
GOOGLE_AI_API_KEY=AI...

# Firebase
NEXT_PUBLIC_FIREBASE_API_KEY=
NEXT_PUBLIC_FIREBASE_AUTH_DOMAIN=
NEXT_PUBLIC_FIREBASE_PROJECT_ID=
NEXT_PUBLIC_FIREBASE_STORAGE_BUCKET=
NEXT_PUBLIC_FIREBASE_MESSAGING_SENDER_ID=
NEXT_PUBLIC_FIREBASE_APP_ID=
FIREBASE_ADMIN_PROJECT_ID=
FIREBASE_ADMIN_CLIENT_EMAIL=
FIREBASE_ADMIN_PRIVATE_KEY=

# Stripe
STRIPE_SECRET_KEY=sk_test_...
NEXT_PUBLIC_STRIPE_PUBLISHABLE_KEY=pk_test_...
STRIPE_WEBHOOK_SECRET=whsec_...

# App
NEXT_PUBLIC_URL=http://localhost:3000
```

## Installation

```bash
# Core dependencies
npm install ai @ai-sdk/react @ai-sdk/openai @ai-sdk/google

# Firebase
npm install firebase firebase-admin

# Stripe
npm install stripe @stripe/stripe-js

# UI
npm install tailwindcss@4 @tailwindcss/postcss

# Testing
npm install -D vitest @vitest/ui @testing-library/react @testing-library/user-event msw

# TypeScript
npm install -D typescript @types/node @types/react tsx
```

## Docker Configuration

```dockerfile
# Dockerfile
FROM node:20-alpine AS base

# Dependencies
FROM base AS deps
WORKDIR /app
COPY package*.json ./
RUN npm ci

# Builder
FROM base AS builder
WORKDIR /app
COPY --from=deps /app/node_modules ./node_modules
COPY . .
RUN npm run build

# Runner
FROM base AS runner
WORKDIR /app
ENV NODE_ENV=production

RUN addgroup --system --gid 1001 nodejs
RUN adduser --system --uid 1001 nextjs

COPY --from=builder /app/public ./public
COPY --from=builder --chown=nextjs:nodejs /app/.next/standalone ./
COPY --from=builder --chown=nextjs:nodejs /app/.next/static ./.next/static

USER nextjs
EXPOSE 3000
ENV PORT=3000

CMD ["node", "server.js"]
```

**Cloud Run Deployment:**

```bash
# Build and deploy
gcloud builds submit --tag gcr.io/PROJECT_ID/research-assistant
gcloud run deploy research-assistant \
  --image gcr.io/PROJECT_ID/research-assistant \
  --platform managed \
  --region us-central1 \
  --allow-unauthenticated \
  --set-env-vars OPENAI_API_KEY=$OPENAI_API_KEY,GOOGLE_AI_API_KEY=$GOOGLE_AI_API_KEY
```

**Confidence:** HIGH (standard Next.js Docker + Cloud Run pattern)

## Key Implementation Decisions

### 1. Multiplexed SSE vs Separate Endpoints

**Decision:** Single multiplexed SSE endpoint

**Rationale:**
- Reduces connection overhead (1 connection vs 2)
- Simplifies client state management (single fetch)
- Enables synchronized streaming (both start together)
- Named events provide clear demultiplexing

**Tradeoff:** More complex server logic, harder to debug

### 2. Server-Side vs Client-Side Stream Orchestration

**Decision:** Server-side orchestration in Route Handler

**Rationale:**
- Protects API keys (never exposed to client)
- Centralized error handling and retry logic
- Easier to implement credit deduction (atomic operation)
- Reduces client bundle size

**Tradeoff:** Server memory usage during concurrent streams

### 3. Firestore vs PostgreSQL for Prompt History

**Decision:** Firestore (pre-selected, but validated)

**Rationale:**
- Already integrated with Firebase Auth
- Real-time listeners for credit balance updates
- Security rules handle row-level security
- Serverless scaling matches Cloud Run
- No connection pooling complexity

**Tradeoff:** Limited query capabilities (no full-text search without Algolia)

### 4. Reconsider Step Implementation

**Pattern: Sequential streaming with context injection**

```typescript
// After initial responses complete
const reconsiderStreams = models.map(async ({ provider, name, peerResponse }) => {
  const result = await streamText({
    model: provider,
    prompt: `Original prompt: ${originalPrompt}

Your initial response: ${yourInitialResponse}

Peer model (${peerName}) responded: ${peerResponse}

Reconsider your response. What did the peer get right or wrong? Revise your answer.`,
    temperature: 0.7,
  });

  return { name, stream: result.textStream };
});
```

**Confidence:** MEDIUM (pattern based on general LLM best practices)

## Version Compatibility Matrix

| Package | Minimum Version | Tested With | Notes |
|---------|----------------|-------------|-------|
| Next.js | 16.0.0 | 16.x | Requires React 19 |
| React | 19.0.0 | 19.x | Concurrent features required |
| ai | 6.0.0 | Latest | v6+ for improved streaming API |
| @ai-sdk/react | 6.0.0 | Latest | Must match ai version |
| Tailwind CSS | 4.0.0 | 4.x | CSS-first config |
| Biome | 2.3.0 | 2.3.x | Specified in requirements |

**Confidence:** MEDIUM (versions from requirements, compatibility assumptions)

## Performance Considerations

### Streaming Latency

| Provider | Model | Time to First Token | Tokens/sec |
|----------|-------|-------------------|------------|
| OpenAI | GPT-5.2 Instant | ~300ms | ~50-80 |
| OpenAI | GPT-5.2 Thinking | ~800ms | ~30-50 |
| Google | Gemini 2.5 Flash | ~200ms | ~60-100 |
| Google | Gemini 3.0 Pro | ~500ms | ~40-70 |

**Confidence:** LOW (estimates based on typical LLM performance, not verified benchmarks)

**Implication:** Gemini Flash starts streaming first in Standard tier, creating perceived performance difference

### Cloud Run Cold Starts

- **Cold start latency:** 1-3 seconds with Next.js 16
- **Mitigation:** Min instances = 1 for production (costs ~$10/month)
- **Alternative:** Warm-up request on user login

### Firestore Read Costs

- **Prompt history page:** 20 reads per page load (1 user doc + 19 prompts)
- **Mitigation:** Client-side caching, pagination with limit(20)
- **Monthly estimate:** 100 users × 10 views/month = 20,000 reads = $0.12/month

## Security Considerations

### API Key Protection

- Server-side only, never in client bundle
- Stored in Cloud Run environment variables
- Rotated quarterly
- Rate limiting per user (100 prompts/hour)

### Credit Fraud Prevention

- Atomic Firestore transactions for credit deduction
- Optimistic locking (check balance before streaming)
- Webhook signature verification for Stripe
- Audit log via CreditTransaction collection

### Firestore Security Rules

```javascript
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {
    match /users/{userId} {
      allow read, write: if request.auth.uid == userId;
    }

    match /prompts/{promptId} {
      allow read: if request.auth.uid == resource.data.userId;
      allow create: if request.auth.uid == request.resource.data.userId
                    && request.resource.data.creditsCharged is number;
      allow update, delete: if false; // Immutable after creation
    }

    match /creditTransactions/{transactionId} {
      allow read: if request.auth.uid == resource.data.userId;
      allow write: if false; // Server-only writes
    }
  }
}
```

**Confidence:** HIGH (standard Firebase security patterns)

## Alternatives Considered

| Category | Selected | Alternative | Why Not |
|----------|----------|-------------|---------|
| LLM SDK | Vercel AI SDK | LangChain | Heavier abstraction, overkill for streaming use case |
| Streaming | Custom SSE | WebSockets | SSE simpler, better browser support, unidirectional sufficient |
| State Management | React useState | Zustand/Redux | Complexity not justified for single-page state |
| Database | Firestore | PostgreSQL | Already integrated, serverless scaling, no connection pooling |
| Hosting | Cloud Run | Vercel | GCP consistency, Docker control, existing GCP billing |

## Gaps & Unknowns

**CRITICAL - Requires verification before Phase 1:**

1. Vercel AI SDK v6+ API changes
   - `streamText()` signature and return type
   - Multi-provider concurrent streaming patterns
   - Error handling and retry mechanisms
   - Token counting and cost tracking

2. Next.js 16 + React 19 streaming gotchas
   - Server Components vs Route Handlers for SSE
   - Suspense boundaries with streaming responses
   - Client-side hydration with partial streams

3. GPT-5.2 and Gemini 3.0 availability
   - Model identifiers (assumed names, not official)
   - Pricing and rate limits
   - Streaming support confirmation

**MEDIUM - Can defer to later phases:**

4. Firestore real-time listeners for credit balance
   - Performance with concurrent users
   - Client-side caching strategy

5. Stripe integration with existing system
   - Credit package SKUs
   - Discount codes and promotions

## Sources

**Note:** All research conducted with training data only due to tool access restrictions. Confidence levels reflect this limitation.

| Source Type | Confidence Impact |
|-------------|------------------|
| Training data (pre-Jan 2025) | LOW-MEDIUM for current APIs |
| Requirements document | HIGH for stack choices |
| Standard patterns | HIGH for general architecture |
| Specific API details | LOW without verification |

**Recommended next step:** Verify Vercel AI SDK v6+ documentation, Next.js 16 SSE patterns, and model availability before starting implementation.
