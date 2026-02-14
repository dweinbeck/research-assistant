# Requirements

## Functional Requirements

### FR-1: Core Comparison Flow
| ID | Requirement | Priority | Phase |
|----|------------|----------|-------|
| FR-1.1 | User can type a prompt in a text input and submit | Must | 1 |
| FR-1.2 | Responses stream in real-time via SSE (not buffered) | Must | 1 |
| FR-1.3 | Two model responses display side-by-side simultaneously | Must | 1 |
| FR-1.4 | User can select Standard or Expert tier via toggle | Must | 1 |
| FR-1.5 | Standard tier uses Gemini 2.5 Flash + GPT-5.2 Instant | Must | 1 |
| FR-1.6 | Expert tier uses Gemini 3.0 Pro + GPT-5.2 Thinking | Must | 1 |

### FR-2: Reconsider Mode
| ID | Requirement | Priority | Phase |
|----|------------|----------|-------|
| FR-2.1 | User can trigger "Reconsider" after both models respond | Must | 3 |
| FR-2.2 | Each model sees peer output and justifies/revises its response | Must | 3 |
| FR-2.3 | Reconsider responses stream in real-time below initial responses | Must | 3 |
| FR-2.4 | Token budget enforcement prevents context window overflow | Must | 3 |

### FR-3: Conversation & Follow-ups
| ID | Requirement | Priority | Phase |
|----|------------|----------|-------|
| FR-3.1 | User can send follow-up prompts that go to both models | Must | 3 |
| FR-3.2 | Follow-ups include prior conversation context | Must | 3 |
| FR-3.3 | Full prompt history persisted across sessions (Firestore) | Must | 3 |
| FR-3.4 | User can view past prompts, responses, and costs | Must | 3 |

### FR-4: Billing & Credits
| ID | Requirement | Priority | Phase |
|----|------------|----------|-------|
| FR-4.1 | Credits debited before LLM calls execute | Must | 1 |
| FR-4.2 | Standard prompt = 10 credits | Must | 1 |
| FR-4.3 | Expert prompt = 20 credits | Must | 1 |
| FR-4.4 | Standard reconsider = +5 credits | Must | 3 |
| FR-4.5 | Expert reconsider = +10 credits | Must | 3 |
| FR-4.6 | Standard follow-up = 5 credits | Must | 3 |
| FR-4.7 | Expert follow-up = 10 credits | Must | 3 |
| FR-4.8 | Tool pricing seeded in billing_tool_pricing collection | Must | 1 |
| FR-4.9 | Integration with existing Stripe credit purchase system | Must | 4 |
| FR-4.10 | Idempotent debit with two-phase commit (PENDING/SUCCESS/FAILED) | Must | 1 |

### FR-5: Authentication
| ID | Requirement | Priority | Phase |
|----|------------|----------|-------|
| FR-5.1 | User must be logged in via Firebase Auth (Google Sign-In) | Must | 1 |
| FR-5.2 | Unauthenticated users redirected to login | Must | 1 |

## Non-Functional Requirements

### NFR-1: Performance
| ID | Requirement | Priority | Phase |
|----|------------|----------|-------|
| NFR-1.1 | Time to first token < 2s (excluding cold start) | Must | 1 |
| NFR-1.2 | Cloud Run min-instances=1 for production | Should | 2 |
| NFR-1.3 | Docker image < 150MB | Should | 2 |
| NFR-1.4 | Streaming timeout with AbortSignal (60s standard, 120s expert) | Must | 2 |

### NFR-2: Reliability
| ID | Requirement | Priority | Phase |
|----|------------|----------|-------|
| NFR-2.1 | One model failure does not break the other model's stream | Must | 1 |
| NFR-2.2 | Per-model error messages (not generic "something went wrong") | Must | 2 |
| NFR-2.3 | Exponential backoff on rate limit (429) responses | Should | 2 |
| NFR-2.4 | Heartbeat events every 15s to detect stalled connections | Should | 2 |

### NFR-3: Observability
| ID | Requirement | Priority | Phase |
|----|------------|----------|-------|
| NFR-3.1 | Structured logging: prompt submissions, model latencies, errors, credit usage | Must | 2 |
| NFR-3.2 | Usage logs in Firestore (research_usage_logs collection) | Must | 2 |

### NFR-4: Security
| ID | Requirement | Priority | Phase |
|----|------------|----------|-------|
| NFR-4.1 | API keys server-side only (never in client bundle) | Must | 1 |
| NFR-4.2 | Cloud Armor rate limiting for research-assistant API routes | Must | 2 |
| NFR-4.3 | Firestore security rules enforce user-scoped access | Must | 4 |

### NFR-5: UX
| ID | Requirement | Priority | Phase |
|----|------------|----------|-------|
| NFR-5.1 | Mobile responsive (stacked layout on < 768px, side-by-side on desktop) | Must | 2 |
| NFR-5.2 | Loading state machine: connecting -> streaming -> complete | Must | 2 |
| NFR-5.3 | Copy button per response | Should | 2 |
| NFR-5.4 | Navy/gold palette (#063970/#C8A55A) with burgundy accent (#8B1E3F) | Must | 1 |

### NFR-6: Infrastructure
| ID | Requirement | Priority | Phase |
|----|------------|----------|-------|
| NFR-6.1 | Dev environment (dev.dan-weinbeck.com) and prod environment (dan-weinbeck.com) | Must | 2 |
| NFR-6.2 | Deployed on GCP Cloud Run (Docker standalone build) | Must | 2 |
| NFR-6.3 | All code lives in personal-brand repo at /apps/research-assistant path patterns | Must | 1 |

## Out of Scope (v1)

- Custom model selection (presets only)
- More than 2 models per prompt
- File/image upload
- Prompt templates or saved prompts
- Separate microservice
- Mobile-first design

---
*Derived from PROJECT.md + research synthesis on 2026-02-13*
