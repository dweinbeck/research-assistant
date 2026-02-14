# Research Assistant

## What This Is

A multi-model LLM comparison tool integrated into dan-weinbeck.com at `/apps/research-assistant`. Users send a prompt and see streaming responses from two AI models side-by-side, with an optional "Reconsider" step where each model sees the other's answer and justifies or revises its response. Aimed at anyone who wants to compare how different models handle the same question before committing to one answer.

## Core Value

Users can instantly compare how two different AI models interpret the same prompt in real time, so they pick the best answer without re-prompting.

## Requirements

### Validated

(None yet -- ship to validate)

### Active

- [ ] User can type a prompt and see streaming responses from two models side-by-side
- [ ] User can choose between Standard tier (Gemini 2.5 Flash + GPT-5.2 Instant) and Expert tier (Gemini 3.0 Pro + GPT-5.2 Thinking)
- [ ] Responses stream in real time via SSE, not buffered
- [ ] User can trigger "Reconsider" after both models respond -- each model sees peer output and justifies/revises
- [ ] User can send follow-up prompts that go to both models (conversational)
- [ ] Each action has a credit cost: Standard prompt=10, Expert prompt=20, Standard reconsider=+5, Expert reconsider=+10, Standard follow-up=5, Expert follow-up=10
- [ ] User must be logged in via Firebase Auth (Google Sign-In) to use the tool
- [ ] Credits are debited from user's billing balance before LLM calls execute
- [ ] User can view full prompt history (past prompts, responses, costs) persisted across sessions
- [ ] Prompt history stored in Firestore with user ID association
- [ ] Tool pricing seeded in billing_tool_pricing Firestore collection
- [ ] Integration with existing Stripe-based credit purchase system
- [ ] Telemetry: structured logging for prompt submissions, model latencies, errors, credit usage
- [ ] Infrastructure: Cloud Armor rate limiting rule for research-assistant API routes
- [ ] Dev environment (dev.dan-weinbeck.com) and prod environment (dan-weinbeck.com) support

### Out of Scope

- Custom model selection (users pick from presets only) -- complexity for v1, revisit if demand
- More than 2 models per prompt -- UI is side-by-side columns, 3+ needs different layout
- File/image upload with prompts -- text-only for v1
- Prompt templates or saved prompts -- build after validating core compare flow
- Separate microservice (API routes live in personal-brand Next.js app) -- LLM API calls are lightweight, no heavy deps
- Mobile-first design -- responsive but desktop-primary for comparison columns

## Context

**Origin:** Port of PromptOS (`~/Documents/prompt-os/`), originally built on AWS (Lambda streaming, DynamoDB, Cognito, CDK). This version rebuilds the same core functionality on GCP using Dan's standard personal-brand stack.

**Stack mapping (AWS -> GCP):**

| PromptOS (AWS) | Research Assistant (GCP) |
|----------------|--------------------------|
| Lambda streaming SSE | Next.js API route + Vercel AI SDK streaming |
| DynamoDB (prompt storage) | Firestore (prompt history) |
| Cognito (auth) | Firebase Auth (Google Sign-In) |
| AWS Secrets Manager | GCP Secret Manager / Cloud Run env vars |
| OpenAI SDK direct | Vercel AI SDK `@ai-sdk/openai` provider |
| Google Generative AI SDK | Vercel AI SDK `@ai-sdk/google` provider |
| CloudFront + S3 (static) | Cloud Run (Next.js standalone) |
| AWS CDK (infra) | Terraform (platform-infra) |
| No billing | Credit-based billing (existing system) |

**Integration point:** Lives in `~/Documents/personal-brand/` repo as Next.js API routes + React components. Uses existing billing, auth, and deployment infrastructure.

**Platform infrastructure:** Global LB with path-based routing managed by `~/Documents/platform-infra/`. Research assistant API routes served by the default personal-brand Cloud Run service (no separate service needed). Cloud Armor rate limiting at the LB level.

**Existing billing system:** Credit-based with Stripe Checkout. 500 credits = $5. 100 free credits on signup. Full Firestore ledger with idempotent debits, refunds, admin adjustments. Brand Scraper uses 50 credits/use. The research assistant introduces tiered per-action pricing.

**Consistent stack across repos (personal-brand, frd-generator, todoist, dave-ramsey):**
- Next.js 16, React 19, Tailwind CSS 4, Biome v2.3
- Firebase Auth + Firestore, GCP Cloud Run
- Vercel AI SDK (`ai` + `@ai-sdk/react`), Vitest

## Constraints

- **Tech stack**: Must use personal-brand stack (Next.js 16, React 19, Tailwind CSS 4, Vercel AI SDK, Firebase, Biome v2.3) -- consistency across repos is non-negotiable
- **Deployment**: GCP Cloud Run via Docker standalone build, dev/prod environments matching platform-infra conventions
- **Billing**: Must integrate with existing credit billing system (Firestore ledger, Stripe) -- no separate payment flow
- **Auth**: Firebase Auth with Google Sign-In -- same as all other tools on the site
- **API keys**: OpenAI and Google AI API keys stored as Cloud Run env vars / Secret Manager -- never in code
- **Models**: Standard tier uses Gemini 2.5 Flash + GPT-5.2 Instant; Expert tier uses Gemini 3.0 Pro + GPT-5.2 Thinking
- **Design**: Navy/gold palette (#063970/#C8A55A) with burgundy accent (#8B1E3F) -- matches personal-brand design system

## Key Decisions

| Decision | Rationale | Outcome |
|----------|-----------|---------|
| Next.js API routes over separate microservice | LLM API calls are lightweight (no Playwright/browser); avoids new repo + Cloud Run service + Terraform route; reuses existing auth/billing middleware | -- Pending |
| Vercel AI SDK over direct OpenAI/Google SDKs | Unified streaming interface, consistent with personal-brand's existing AI assistant pattern, provider-agnostic | -- Pending |
| Tiered pricing (Standard/Expert) over flat rate | Different model costs justify different credit prices; gives users cost control | -- Pending |
| Per-action pricing over per-session | Follow-ups and reconsider are optional; charging per action is fairer and more transparent | -- Pending |
| Firestore over PostgreSQL for history | Consistent with personal-brand's existing data layer; no need for relational queries on prompt history | -- Pending |
| Side-by-side columns over tabbed view | Core differentiator is simultaneous comparison; tabs defeat the purpose | -- Pending |

---
*Last updated: 2026-02-13 after initialization*
