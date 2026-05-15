# CLAUDE.md — Project Architecture & Build Specification (v2)

> **Working name:** `arcadia` (placeholder — replace with final product name throughout).
> **One-line description:** A multi-tenant agent-native platform that takes a B2B sales company's ICP and brand, finds leads, researches each target account, generates a personalized pitch deck and email, sends it, handles replies, and books meetings — end to end.
> **Status:** Specification. No code written yet.
> **Owner:** Flavio Elia (product), CTO TBD (engineering).
> **Mode:** Internal-only for Phases 0–4. Commercialization gated on Phase 5 decision.

This document is the source of truth for Claude Code and any engineer working on this codebase. Every architectural decision below is **locked** unless this document is updated. When in doubt, this document wins.

---

## 1. Product Specification

### 1.1 What we are building

A platform for any B2B sales company. Tenant signs up, tells the platform about their company in 5 minutes via a conversational onboarding, and the platform's agent crew handles outbound sales end-to-end: prospecting, research, strategy, bespoke deck generation, personalized email, reply handling, meeting booking.

The platform is **industry-agnostic**. Brand voice, ICP, offering, and deck template are all extracted from each tenant's inputs at onboarding. No hardcoded industry logic.

The same engine is exposed four ways:

- **Web app** (`app.<domain>`) — the operator's home. Onboarding, dashboard, lead management, settings, API keys.
- **REST API** — for developers integrating into their applications.
- **MCP server** — for AI agents (Claude, ChatGPT, Cursor, Claude Code, custom) to install our capabilities as native tools.
- **A2A peer** — for other agent platforms (LangGraph, CrewAI, AutoGen, Google ADK) to discover and delegate tasks to our agents.

Optional human-channel integrations (tenant-opt-in, Phase 4+): Telegram, Slack (future), WhatsApp (future).

### 1.2 The agent crew (12 agents)

Each agent has one responsibility. Agents communicate via events on the bus, not direct calls.

| Agent | Responsibility |
|---|---|
| **Onboarder** | Extract brand voice, ICP, offering from tenant inputs at signup |
| **Prospector** | Discover leads matching tenant's ICP |
| **Orchestrator** | Owns lead state machine; decides which agent acts next |
| **Intake** | Accepts and normalizes leads from any source |
| **Researcher** | Deep research on a company + person |
| **Strategist** | Maps tenant offering to prospect; defines angle |
| **Deck Builder** | Generates bespoke pitch deck from master template |
| **Copywriter** | Drafts email referencing research and deck |
| **Sentinel** | Compliance + quality + brand gate before send |
| **Sender** | Sends via tenant's connected mailbox |
| **Reply Handler** | Classifies replies, routes to next action |
| **Closer** | Books meetings via tenant's calendar |

Plus optionally a **Concierge** agent in Phase 4+ if Telegram/Slack/WhatsApp integration is built — same agent behind any chat transport.

### 1.3 What we are NOT building

- Email deliverability infrastructure. Tenants connect their own Gmail/Outlook (v1) or Smartlead/Instantly (v2).
- CRM. We sync to tenant's CRM via Nango integrations.
- Prospecting databases. We integrate Apollo, Clay, etc. as tools.
- Calendar booking engine. We integrate Cal.com, Calendly, Google Calendar.
- AI inference. We call Anthropic API (primary) and OpenAI API (fallback).
- **Marketing website.** App + docs only. Marketing site deferred until commercialization.

### 1.4 Operating mode

**Phases 0–4 (months 0–9): internal-only.** No public signup, no billing, no marketing site. Elchai Group's 5 brands (Nodalum, CAV, OpenReal, Peptalys, TUE) onboarded as separate tenants. 3–5 friendly external tenants of varied industries onboarded by invite in Phase 2+ to validate the generic-platform claim.

**Phase 5 (months 9–12, gated): commercialization decision.** Only if internal usage data validates product-market fit, add billing, public signup, pricing page.

---

## 2. Technology Stack (LOCKED)

Any deviation requires an ADR in `/docs/adr/`.

### 2.1 Languages

- **Python 3.12** — agent logic, API service, background workers.
- **TypeScript 5.x on Node.js 20 LTS** — frontend, type-safe SDK.
- **SQL** — Postgres dialect.

### 2.2 Backend frameworks

| Layer | Choice | Version | Why |
|---|---|---|---|
| Agent SDK | Anthropic Claude Agent SDK | latest | Native Claude, Memory, sub-agents |
| Multi-agent orchestration | LangGraph | `>=0.4` | State machines, checkpoints, HITL |
| MCP server | FastMCP | latest | Standard for tool exposure |
| A2A | `a2a-python` (Google) | latest | Agent-to-agent protocol |
| REST API | FastAPI | latest | OpenAPI auto-gen, async |

### 2.3 Frontend stack

| Layer | Choice | Why |
|---|---|---|
| Framework | Next.js 15 (App Router) | RSC, streaming |
| Styling | Tailwind 4 | Standard |
| Components | shadcn/ui | Copyable, customizable |
| Forms | React Hook Form + Zod | Type-safe forms |
| Data fetching | TanStack Query + tRPC | Type-safe API client |
| Auth UI | Clerk components | Plug-and-play |
| Charts | Recharts | Standard |
| Tables | TanStack Table | Standard |
| Realtime | Server-Sent Events from FastAPI | Push state changes to UI |
| Deck preview | React-PDF | Inline PDF viewer |

### 2.4 Data + state

| Layer | Choice |
|---|---|
| Primary DB | Postgres 16 on Neon |
| Vector store | pgvector inside same Postgres |
| Object storage | Cloudflare R2 |
| Cache | Upstash Redis |
| Event bus | Redpanda Cloud |

### 2.5 Execution + orchestration

| Layer | Choice |
|---|---|
| Durable workflows | Inngest |
| Background jobs | Inngest functions |
| Scheduling | Inngest cron |

### 2.6 Auth + tenancy + integrations

| Layer | Choice |
|---|---|
| Tenant auth + orgs | Clerk |
| API key auth | Custom on top of Clerk orgs (test + live modes) |
| OAuth integrations | Nango |

### 2.7 Observability

| Layer | Choice |
|---|---|
| LLM tracing | Langfuse (self-hosted) |
| App tracing | OpenTelemetry → Honeycomb |
| Metrics | Prometheus → Grafana Cloud |
| Errors | Sentry |
| Logs | Better Stack |
| Status page | Instatus (Phase 5 only) |

### 2.8 Billing (Phase 5 only)

| Layer | Choice |
|---|---|
| Payments | Stripe |
| Usage metering | Lago |

### 2.9 Documentation

Mintlify at `docs.<domain>`.

### 2.10 Hosting

| Layer | Phase 0–4 | Phase 5+ |
|---|---|---|
| API + workers | Railway | AWS ECS/GKE |
| App (Next.js) | Vercel | Vercel |
| Docs (Mintlify) | Vercel | Vercel |

### 2.11 Tooling

- Package management: `uv` (Python), `pnpm` (Node).
- Monorepo: `turborepo` for TS, `uv workspaces` for Python.
- Linting: `ruff` + `mypy --strict` (Python), `eslint` + `prettier` + `tsc --strict` (TS).
- Testing: `pytest` + `httpx`, `vitest`, `playwright`.
- CI: GitHub Actions.
- Secrets: Doppler or Infisical.

---

## 3. System Architecture

### 3.1 High-level flow

```
TENANT OPERATOR ──┐
                  ├──► app.<domain> (Next.js)
TEAMMATES ────────┘            │
                               ▼
DEVELOPER APPS ───► REST API ──┤
AI ASSISTANTS ────► MCP server ┼─► API Gateway (auth, tenant, rate limit, idempotency)
OTHER AGENTS ─────► A2A peer ──┘            │
TELEGRAM (opt-in) ► webhook ────────────────►│
                                             ▼
                            Capability Service (FastAPI + FastMCP + a2a-python)
                                             │
                                       Same agent classes
                                       behind every transport
                                             │
                                             ▼
                                     Inngest Functions
                                    (durable agent runs)
                                             │
                                             ▼
                                  LangGraph state machines
                                    (one per active lead)
                                             │
                                             ▼
                             Agents ─► Anthropic API
                                    ─► Postgres + pgvector
                                    ─► R2 (artifacts)
                                    ─► Nango (tenant integrations)
                                    ─► Web search, Apollo, etc.
                                             │
                                             ▼
                                    Events → Redpanda
                                             │
                              Webhooks delivered to tenants
                              Realtime updates to app via SSE
```

### 3.2 One agent, four transports

Each agent is implemented once and exposed four ways. **Never duplicate logic across transports.**

```python
# packages/agents/researcher/agent.py
class ResearcherAgent:
    async def research_company(self, domain: str, tenant_id: str) -> ResearchBrief:
        ...

# REST adapter
@router.post("/v1/research/company")
async def rest(req, tenant=Depends(get_tenant)):
    return await researcher.research_company(req.domain, tenant.id)

# MCP adapter
@mcp.tool()
async def research_company(domain, ctx):
    return (await researcher.research_company(domain, ctx.session.tenant_id)).model_dump()

# A2A adapter
@a2a_skill(name="research_company")
async def a2a(task):
    return A2AArtifact(content=(await researcher.research_company(task.input["domain"], task.context.tenant_id)).model_dump())

# App: calls REST through tRPC client (no separate adapter)
```

### 3.3 Sync vs async endpoints

| Latency | Pattern | Examples |
|---|---|---|
| < 2s | Sync | classify reply, voice extract |
| 2–30s | Sync | research, generate email |
| > 30s | Async with job_id | generate deck, campaign run |

Async endpoints return `{ job_id, status: "queued", poll_url, webhook_subscribed }`.

---

## 4. Multi-Tenant Isolation (CRITICAL — NEVER VIOLATE)

Five isolation layers. Each enforces tenant_id independently.

### 4.1 Database rows — Postgres RLS

Every table has `tenant_id UUID NOT NULL` and an RLS policy:

```sql
CREATE POLICY tenant_isolation ON contacts
    FOR ALL USING (tenant_id = current_setting('app.current_tenant')::uuid);
ALTER TABLE contacts ENABLE ROW LEVEL SECURITY;
```

API gateway sets `SET app.current_tenant = '...'` on every session before tenant queries. **No queries that bypass RLS.**

### 4.2 Vector embeddings — namespace per tenant

```sql
CREATE TABLE embeddings (
    tenant_id UUID NOT NULL,
    namespace TEXT NOT NULL,  -- 'tenant_<uuid>_<source_type>'
    embedding vector(1536),
    ...
);
```

Every similarity query includes namespace filter. Cross-tenant search forbidden.

### 4.3 Object storage — workspace per tenant

```
r2://arcadia-artifacts/tenants/<tenant_id>/
    decks/<lead_id>/{deck.pptx, deck.pdf}
    brand/{master_deck.pptx, logo.png}
    research/<company_id>/research_brief.json
```

Signed URLs scoped to single tenant, expiry < 1 hour.

### 4.4 Agent context + memory

Every Claude Agent SDK memory instance, LangGraph state object, and prompt cache key includes tenant_id.

### 4.5 Cost attribution

Every LLM call logged at the moment of call with `{tenant_id, agent, model, input_tokens, output_tokens, cost_usd, timestamp}`. **Direct `anthropic.Anthropic()` calls outside `arcadia_core/llm/client.py` are forbidden.**

---

## 5. Data Model

Schema in `/apps/api/migrations/`. Alembic.

```sql
-- Tenancy
CREATE TABLE organizations (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    clerk_org_id TEXT UNIQUE NOT NULL,
    name TEXT NOT NULL,
    plan TEXT NOT NULL DEFAULT 'internal',
    onboarding_state TEXT NOT NULL DEFAULT 'pending',
    created_at TIMESTAMPTZ DEFAULT NOW(),
    settings JSONB DEFAULT '{}'::jsonb
);

CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    clerk_user_id TEXT UNIQUE NOT NULL,
    org_id UUID NOT NULL REFERENCES organizations(id),
    email TEXT NOT NULL,
    role TEXT NOT NULL DEFAULT 'member',
    created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE api_keys (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES organizations(id),
    key_hash TEXT NOT NULL UNIQUE,
    prefix TEXT NOT NULL,
    mode TEXT NOT NULL CHECK (mode IN ('test', 'live')),
    scopes TEXT[] NOT NULL DEFAULT '{}',
    last_used_at TIMESTAMPTZ,
    revoked_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ DEFAULT NOW()
);

-- Per-tenant config
CREATE TABLE brand_voices (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES organizations(id),
    tone JSONB NOT NULL,
    vocabulary JSONB NOT NULL,
    sample_content TEXT[],
    source_url TEXT,
    updated_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE icp_profiles (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES organizations(id),
    name TEXT NOT NULL,
    firmographics JSONB NOT NULL,
    personas JSONB NOT NULL,
    disqualifiers JSONB DEFAULT '[]'::jsonb
);

CREATE TABLE offerings (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES organizations(id),
    name TEXT NOT NULL,
    description TEXT NOT NULL,
    value_props JSONB NOT NULL,
    pricing JSONB,
    target_icp_id UUID REFERENCES icp_profiles(id),
    master_deck_r2_key TEXT,
    created_at TIMESTAMPTZ DEFAULT NOW()
);

-- Contact graph
CREATE TABLE companies (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES organizations(id),
    domain TEXT NOT NULL,
    name TEXT,
    enriched_data JSONB DEFAULT '{}'::jsonb,
    research_brief_r2_key TEXT,
    last_researched_at TIMESTAMPTZ,
    UNIQUE (tenant_id, domain)
);

CREATE TABLE contacts (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES organizations(id),
    company_id UUID REFERENCES companies(id),
    email TEXT,
    linkedin_url TEXT,
    full_name TEXT,
    title TEXT,
    enriched_data JSONB DEFAULT '{}'::jsonb,
    consent_status TEXT DEFAULT 'unknown',
    consent_basis TEXT,
    UNIQUE (tenant_id, email)
);

-- Leads
CREATE TABLE leads (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES organizations(id),
    contact_id UUID NOT NULL REFERENCES contacts(id),
    company_id UUID NOT NULL REFERENCES companies(id),
    offering_id UUID NOT NULL REFERENCES offerings(id),
    state TEXT NOT NULL DEFAULT 'received',
    source TEXT NOT NULL,
    source_metadata JSONB,
    strategy JSONB,
    deck_r2_key TEXT,
    last_event_at TIMESTAMPTZ DEFAULT NOW(),
    created_at TIMESTAMPTZ DEFAULT NOW()
);
CREATE INDEX idx_leads_tenant_state ON leads(tenant_id, state);

-- Email
CREATE TABLE email_threads (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES organizations(id),
    lead_id UUID NOT NULL REFERENCES leads(id),
    provider_thread_id TEXT,
    subject TEXT
);

CREATE TABLE email_messages (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES organizations(id),
    thread_id UUID NOT NULL REFERENCES email_threads(id),
    direction TEXT NOT NULL CHECK (direction IN ('outbound', 'inbound')),
    body_text TEXT,
    body_html TEXT,
    sent_at TIMESTAMPTZ,
    classification TEXT,
    classification_confidence REAL,
    metadata JSONB
);

-- Agent runs + LLM cost
CREATE TABLE agent_runs (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES organizations(id),
    agent_name TEXT NOT NULL,
    lead_id UUID REFERENCES leads(id),
    inngest_run_id TEXT,
    state JSONB NOT NULL,
    started_at TIMESTAMPTZ DEFAULT NOW(),
    completed_at TIMESTAMPTZ,
    status TEXT NOT NULL DEFAULT 'running',
    error JSONB
);

CREATE TABLE llm_calls (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES organizations(id),
    agent_run_id UUID REFERENCES agent_runs(id),
    provider TEXT NOT NULL,
    model TEXT NOT NULL,
    input_tokens INT NOT NULL,
    output_tokens INT NOT NULL,
    cost_usd NUMERIC(10, 6) NOT NULL,
    latency_ms INT,
    created_at TIMESTAMPTZ DEFAULT NOW()
);
CREATE INDEX idx_llm_calls_tenant_time ON llm_calls(tenant_id, created_at);

-- Events
CREATE TABLE events (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES organizations(id),
    type TEXT NOT NULL,
    actor TEXT NOT NULL,
    subject_type TEXT,
    subject_id UUID,
    payload JSONB NOT NULL,
    created_at TIMESTAMPTZ DEFAULT NOW()
);
CREATE INDEX idx_events_tenant_time ON events(tenant_id, created_at DESC);

-- Integrations
CREATE TABLE integrations (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES organizations(id),
    provider TEXT NOT NULL,
    nango_connection_id TEXT NOT NULL,
    status TEXT NOT NULL DEFAULT 'active',
    metadata JSONB DEFAULT '{}'::jsonb,
    UNIQUE (tenant_id, provider)
);

-- Suppression
CREATE TABLE suppression_list (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES organizations(id),
    email TEXT,
    domain TEXT,
    reason TEXT NOT NULL,
    CHECK (email IS NOT NULL OR domain IS NOT NULL)
);

-- Embeddings
CREATE TABLE embeddings (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES organizations(id),
    namespace TEXT NOT NULL,
    source_type TEXT NOT NULL,
    source_id UUID,
    content TEXT NOT NULL,
    embedding vector(1536) NOT NULL,
    metadata JSONB DEFAULT '{}'::jsonb
);

-- Webhooks
CREATE TABLE webhook_endpoints (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES organizations(id),
    url TEXT NOT NULL,
    secret TEXT NOT NULL,
    subscribed_events TEXT[] NOT NULL,
    status TEXT NOT NULL DEFAULT 'active'
);

CREATE TABLE webhook_deliveries (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    endpoint_id UUID NOT NULL REFERENCES webhook_endpoints(id),
    event_id UUID NOT NULL REFERENCES events(id),
    attempts INT NOT NULL DEFAULT 0,
    last_attempt_at TIMESTAMPTZ,
    delivered_at TIMESTAMPTZ,
    last_status_code INT
);

-- Idempotency
CREATE TABLE idempotency_keys (
    key TEXT PRIMARY KEY,
    tenant_id UUID NOT NULL REFERENCES organizations(id),
    request_hash TEXT NOT NULL,
    response JSONB,
    status_code INT,
    expires_at TIMESTAMPTZ NOT NULL
);

-- Telegram (Phase 4+, optional per tenant)
CREATE TABLE telegram_integrations (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES organizations(id),
    bot_token_encrypted TEXT NOT NULL,
    bot_username TEXT NOT NULL,
    bot_id BIGINT NOT NULL,
    webhook_secret TEXT NOT NULL,
    status TEXT NOT NULL DEFAULT 'active',
    settings JSONB DEFAULT '{}'::jsonb,
    UNIQUE (tenant_id)
);

CREATE TABLE telegram_users (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES organizations(id),
    telegram_user_id BIGINT NOT NULL,
    user_id UUID REFERENCES users(id),
    role TEXT NOT NULL DEFAULT 'member',
    linked_at TIMESTAMPTZ DEFAULT NOW(),
    UNIQUE (tenant_id, telegram_user_id)
);
```

Every table has RLS enabled with the standard `tenant_isolation` policy.

---

## 6. Agent Specifications

Each agent lives in `/packages/agents/<agent_name>/`. Each has `agent.py`, `prompts/` (system prompts as `.md`), `tools.py`, `schemas.py`, `tests/`.

### 6.1 Onboarder

**Purpose:** Extract tenant's brand voice, ICP, offering from minimal inputs during signup.
**Input:** `{ company_url, optional_clarifications? }`
**Output:** Draft `BrandVoice`, `IcpProfile`, `Offering` records.
**Tools:** `web_fetch`, `web_search`, `parse_pptx`.
**Behavior:** Reads tenant's website (home, about, products, customers, case studies, pricing). Identifies what they sell, who they sell to, brand tone, sample language. Asks 1–3 clarifying questions only if extraction is ambiguous.
**Model:** Sonnet.

### 6.2 Prospector

**Purpose:** Find leads matching tenant's ICP.
**Input:** `{ icp_profile_id, volume, exclusions? }`.
**Output:** Scored list of leads (companies + persons).
**Tools:** Apollo, LinkedIn Sales Navigator (via Nango), Crunchbase, web search.
**Behavior:** Generates target queries from ICP firmographics. Identifies target persona per company. Deduplicates against existing pipeline. Filters suppression list + exclusions.
**Model:** Sonnet for query generation; deterministic dedup.

### 6.3 Researcher

**Purpose:** Structured research brief on a target company.
**Input:** `{ domain, person_email?, depth: "quick"|"standard"|"deep" }`.
**Output:** `ResearchBrief` (company facts, signals, pain points, decision-makers, fit assessment, sources).
**Tools:** `web_search`, `web_fetch`, `fetch_linkedin_company`, `query_company_database`.
**Model:** Sonnet for quick/standard, Opus for deep.
**Error handling:** If insufficient info, `confidence: low`, `requires_human_input: true`. Never hallucinate.

### 6.4 Strategist

**Input:** `ResearchBrief`, `Offering`, `IcpProfile`, `BrandVoice`.
**Output:** `CampaignStrategy` (target_persona, hook, value_prop_framing, talking_points, anticipated_objections, recommended_cta).
**Model:** Sonnet.

### 6.5 Deck Builder

**Input:** `CampaignStrategy`, `ResearchBrief`, `master_deck_r2_key`.
**Output:** `{ pptx_url, pdf_url, slide_summaries }`.
**Implementation:**
1. Load master deck via `python-pptx`.
2. Parse placeholders per slide.
3. Generate company-specific content per slide.
4. Substitute content, swap in company logo (Clearbit Logo API).
5. Save PPTX, render PDF via headless LibreOffice.
6. Upload to R2, return signed URLs.

**Cost target:** < $1 per deck.
**Model:** Sonnet per slide.

### 6.6 Copywriter

**Input:** `CampaignStrategy`, `ResearchBrief`, `deck_url`, `BrandVoice`, `previous_messages?`.
**Output:** `{ subject, body_text, body_html, references_used }`.
**Constraint:** Every email MUST reference ≥2 specific facts from `ResearchBrief.sources`. Sentinel rejects otherwise.
**Model:** Sonnet.

### 6.7 Sentinel

**Checks (fail-fast):**
1. Suppression list (SQL).
2. Consent state (GDPR).
3. Frequency cap.
4. Fact grounding (verification prompt).
5. Brand compliance.
6. Quality threshold (judge-LLM).
7. Prohibited content.

**Models:** Haiku for checks; Sonnet for rejection reasons.

### 6.8 Sender

Sends via tenant's mailbox (Nango → Gmail/Outlook). Adds `List-Unsubscribe` header (RFC 8058). Captures message-id + thread-id. Sets up reply monitoring via Gmail push.

### 6.9 Reply Handler

**Classes:** interested, objection, question, out_of_office, not_interested, unsubscribe, wrong_person, auto_reply, other.
**Routing:** interested → Closer. objection/question → Copywriter. unsubscribe → suppression. wrong_person → Copywriter for redirect. OOO → snooze. not_interested → close.
**Model:** Haiku.

### 6.10 Closer

Max 4 back-and-forth before mandatory human escalation. Escalates on: pricing, technical-out-of-competence, enterprise procurement. Books via Cal.com/Calendly.
**Model:** Sonnet.

### 6.11 Intake

Validates lead. Deduplicates by email + domain+name. Enriches via Apollo if connected. Fit-screens against ICP — below threshold → `state = disqualified`.

### 6.12 Orchestrator

LangGraph state machine. Each transition is a graph node. Checkpointed to Postgres. Triggered by events.

```python
graph = StateGraph(LeadState)
graph.add_node("intake", intake_node)
graph.add_node("research", research_node)
graph.add_node("strategize", strategize_node)
graph.add_node("build_deck", deck_node)
graph.add_node("copywrite", copywriter_node)
graph.add_node("sentinel", sentinel_node)
graph.add_node("send", sender_node)
graph.add_node("wait_for_reply", wait_node)
graph.add_node("handle_reply", reply_handler_node)
graph.add_node("close", closer_node)
graph.add_conditional_edges("sentinel", route_sentinel)
graph.add_conditional_edges("handle_reply", route_reply)
```

---

## 7. API Surface

OpenAPI spec auto-generated at `/apps/api/openapi.yaml`.

### 7.1 Versioning

All endpoints under `/v1/`. Breaking changes → `/v2/`. Old versions supported 12 months after deprecation.

### 7.2 Auth

`Authorization: Bearer <api_key>`. Keys prefixed `arc_test_` or `arc_live_`. Test keys hit sandbox (no real sends/bookings).

### 7.3 Endpoints (v1)

**Atomic (sync):**
```
POST /v1/research/company
POST /v1/research/person
POST /v1/generate/email
POST /v1/classify/reply
POST /v1/voice/extract
```

**Generation (async):**
```
POST /v1/generate/deck         → { job_id, status_url }
POST /v1/sequence/generate     → { job_id, status_url }
```

**Workflows (async):**
```
POST /v1/campaign/run          → { job_id, status_url }
POST /v1/inbound/qualify       → { job_id | result }
```

**Stateful pipeline:**
```
POST   /v1/leads               → add lead(s)
GET    /v1/leads/{id}          → state + history
DELETE /v1/leads/{id}          → close
POST   /v1/leads/{id}/replay   → retry from last successful state
```

**Jobs:**
```
GET /v1/jobs/{id}              → { status, progress, result?, error? }
```

**Webhooks:**
```
GET    /v1/webhooks
POST   /v1/webhooks
DELETE /v1/webhooks/{id}
POST   /v1/webhooks/{id}/test
```

**Tenant config:**
```
GET  /v1/tenants/me/brand-voice
PUT  /v1/tenants/me/brand-voice
GET  /v1/tenants/me/offerings
POST /v1/tenants/me/offerings
PUT  /v1/tenants/me/master-deck
GET  /v1/tenants/me/icp-profiles
POST /v1/tenants/me/icp-profiles
GET  /v1/tenants/me/integrations
```

### 7.4 Idempotency

Every POST accepts `Idempotency-Key: <uuid>`. Server caches key→response for 24h.

### 7.5 MCP capability map

Same capabilities at `https://mcp.<domain>/v1`. Auth via API key in MCP session init.
Tools exposed: `research_company`, `research_person`, `generate_email`, `generate_deck`, `classify_reply`, `run_campaign`, `get_job`, `add_lead`, `get_lead`.

### 7.6 A2A agent card

Published at `https://a2a.<domain>/.well-known/agent.json`. Each capability published as A2A skill.

---

## 8. Frontend Architecture

### 8.1 Two web surfaces

| Surface | URL | Audience | Auth |
|---|---|---|---|
| **App** | `app.<domain>` | Tenant operators + teams | Clerk |
| **Docs** | `docs.<domain>` | Developers integrating | Public (Mintlify) |

No marketing site at this stage. The signup flow at `app.<domain>` is the entry point.

### 8.2 App routes

```
/
├── /sign-in                  (Clerk)
├── /sign-up                  (Clerk)
├── /onboarding              ← conversational onboarding flow
│   ├── /step/company
│   ├── /step/confirm
│   ├── /step/deck
│   ├── /step/integrations
│   └── /step/preview        ← magic moment
├── /                         ← dashboard home
├── /leads
│   └── /[lead_id]           ← lead detail page
├── /pipeline                 ← kanban view
├── /campaigns
│   └── /[campaign_id]
├── /insights                 ← analytics
└── /settings
    ├── /brand-voice
    ├── /icp
    ├── /offerings
    ├── /master-deck
    ├── /integrations
    ├── /api-keys
    ├── /team
    ├── /telegram             ← Phase 4+, optional
    └── /billing              ← Phase 5+ only
```

### 8.3 Onboarding flow

**Step 0 — Account creation.** Clerk: email/password or SSO. Lands at `/onboarding`.

**Step 1 — Tell us about your company.** Single input: company URL. Onboarder agent runs (15–30s). Progressive loading messages.

**Step 2 — Confirm what we learned.** Editable view of extracted brand voice, ICP, offering. Tenant confirms or edits inline.

**Step 3 — Upload master deck.** Drag-drop PPTX. Deck Builder pre-parses placeholders. Preview. Skippable (generic fallback).

**Step 4 — Connect email + calendar.** Nango OAuth popups. Gmail/Outlook + Cal.com/Calendly/Google Calendar.

**Step 5 — Preview moment.** "Type any company you'd love to win." User types domain. Full pipeline runs (research + strategy + deck + email). 60–90s. Result shown: email preview + embedded deck preview + research brief. **This is the close.**

**Step 6 — Done.** API key auto-generated. MCP + A2A URLs visible. CTA to dashboard.

### 8.4 Lead detail page

Most important post-onboarding screen. Shows:

- Lead summary header (company, contact, state, source, created)
- Research brief (collapsible, structured)
- Strategy (collapsible)
- Generated deck (embedded preview + PPTX/PDF download)
- Email thread (timeline, classification badges on replies)
- Agent run log (collapsible: every step, model, cost, timing)
- Actions panel (approve, pause, regenerate, manually take over, close)

### 8.5 Settings → API keys page

```
API KEYS                                            [+ Create new key]

Test (sandbox)
arc_test_4Bn...3Kj    Created May 10    Last used: 2h ago    [Reveal][Revoke]

Live
arc_live_8Fp...9Rm    Created May 10    Last used: 5m ago    [Reveal][Revoke]

MCP ENDPOINT
https://mcp.<domain>/v1/<tenant-id>                          [Copy]

A2A AGENT CARD
https://a2a.<domain>/.well-known/agent.json                  [Copy]
```

New key shown once in modal with copy button + warning. After modal closes: prefix + last-4 only.

### 8.6 Realtime updates

Server-Sent Events from FastAPI push lead state changes to the app. User sees "Researching... → Strategizing... → Building deck... → Done" live. No polling.

### 8.7 Internal admin app (Phase 2+)

Separate domain `admin.<domain>`. For team to view all tenants, troubleshoot, view all agent runs, suspend abusive tenants. Different Clerk auth scope.

---

## 9. Event Bus

### 9.1 Topics

Single topic `agent.events` with `tenant_id` as partition key. Enterprise tenants get dedicated topics on request.

### 9.2 Event shape

```json
{
  "id": "evt_...",
  "tenant_id": "...",
  "type": "lead.researched",
  "actor": "researcher_agent",
  "subject": { "type": "lead", "id": "..." },
  "payload": {},
  "trace_id": "...",
  "created_at": "..."
}
```

### 9.3 Event catalog

Lead: `lead.received`, `lead.researched`, `lead.strategized`, `lead.deck_ready`, `lead.email_drafted`, `lead.email_approved`, `lead.email_sent`, `lead.replied`, `lead.qualified`, `lead.booked`, `lead.lost`, `lead.unsubscribed`.

Agent: `agent.started`, `agent.completed`, `agent.failed`, `agent.escalated`.

Tenant: `tenant.created`, `tenant.onboarding_completed`, `tenant.integration_connected`.

Usage: `usage.recorded`, `quota.threshold_reached`.

### 9.4 Webhook delivery

Events matching tenant subscriptions queued for HMAC-signed delivery. Retry 5x exponential backoff.

---

## 10. Project Structure

```
arcadia/
├── CLAUDE.md
├── README.md
├── docs/
│   ├── adr/
│   ├── runbooks/
│   └── domain-glossary.md
├── apps/
│   ├── api/                          # FastAPI + FastMCP + a2a-python
│   │   ├── pyproject.toml
│   │   ├── src/arcadia_api/
│   │   │   ├── main.py
│   │   │   ├── mcp_server.py
│   │   │   ├── a2a_server.py
│   │   │   ├── middleware/
│   │   │   ├── routes/
│   │   │   ├── adapters/
│   │   │   └── deps.py
│   │   ├── migrations/
│   │   └── openapi.yaml
│   ├── app/                          # Next.js tenant app
│   │   ├── package.json
│   │   ├── app/                      # App Router
│   │   ├── components/
│   │   └── lib/
│   ├── docs/                         # Mintlify
│   ├── admin/                        # Internal admin (Phase 2+)
│   └── workers/                      # Inngest functions
├── packages/
│   ├── agents/                       # the agent crew
│   │   ├── src/arcadia_agents/
│   │   │   ├── onboarder/
│   │   │   ├── prospector/
│   │   │   ├── orchestrator/
│   │   │   ├── intake/
│   │   │   ├── researcher/
│   │   │   ├── strategist/
│   │   │   ├── deck_builder/
│   │   │   ├── copywriter/
│   │   │   ├── sentinel/
│   │   │   ├── sender/
│   │   │   ├── reply_handler/
│   │   │   ├── closer/
│   │   │   ├── common/
│   │   │   └── tools/
│   ├── core/                         # shared schemas, DB, events, observability
│   │   └── src/arcadia_core/
│   │       ├── db/
│   │       ├── events/
│   │       ├── nango/
│   │       ├── llm/                  # LLM client wrapper (cost logging)
│   │       ├── observability/
│   │       └── schemas/
│   ├── ui/                           # shared UI components
│   ├── sdk-python/
│   └── sdk-typescript/
├── infra/
│   ├── terraform/
│   └── docker/
├── scripts/
├── pyproject.toml
├── package.json
├── turbo.json
└── docker-compose.yml
```

---

## 11. Coding Conventions

### 11.1 Python

- Python 3.12. Modern syntax.
- All async I/O.
- Pydantic v2 for all schemas.
- SQLAlchemy 2.x async.
- `mypy --strict` is CI-blocking.
- `ruff` with `select = ["ALL"]` minus documented ignores.
- No `print()` — use `structlog`.

### 11.2 TypeScript

- Strict mode + `noUncheckedIndexedAccess`.
- No `any`.
- Server Components by default.
- Zod for runtime validation.

### 11.3 Prompts

- Every system prompt is a `.md` file in agent's `prompts/`.
- Never inline prompts beyond one-liner templates.
- Prompts versioned in Langfuse.

### 11.4 Tests

- Unit (mocked LLM): 70%.
- Integration (real DB + Inngest, mocked LLM): 20%.
- Eval (real LLM, golden dataset): 8%.
- E2E (Playwright): 2%.
- Coverage target: 80% on non-prompt code.

### 11.5 Logging

Every log includes `tenant_id`, `trace_id`, `agent_name`. Every LLM call traced to Langfuse.

### 11.6 Error handling

- Domain errors typed.
- Never log secrets or PII.
- Never swallow exceptions.

---

## 12. Critical Rules (NEVER VIOLATE)

Enforced by CI and code review.

1. **`tenant_id` in every query.** RLS or explicit `WHERE tenant_id = ?`.
2. **Vector namespace per tenant.** Cross-tenant similarity forbidden.
3. **R2 keys are tenant-prefixed.** Every key starts with `tenants/<tenant_id>/`.
4. **All LLM calls go through `arcadia_core/llm/client.py`.** Direct SDK calls forbidden.
5. **No PII in logs.** Use IDs.
6. **Idempotency keys on all mutating POSTs.**
7. **All long-running operations via Inngest.** No `asyncio.create_task` for work needing to survive deploys.
8. **Prompts in `.md` files, versioned in Langfuse.**
9. **Schema-first.** Typed input/output everywhere.
10. **One adapter per transport.** Same agent logic powers REST, MCP, A2A. No branching on transport in agent code.
11. **Parameterized queries only.**
12. **Webhook payloads HMAC-signed.**
13. **Test mode keys never touch production integrations.**
14. **LLM outputs validated with Pydantic.** Retry once on schema violation, then fail typed.
15. **Suppression list consulted before every send.** Sentinel cannot be bypassed.
16. **No assumption about tenant industry.** Industry is detected, not hardcoded.
17. **No marketing site or public-facing content in Phase 0–4.** Stay invite-only.

---

## 13. Build Sequence

### Phase 0 — Skeleton (week 1)

- Monorepo bootstrap.
- One agent (Researcher) end-to-end.
- Exposed via REST + MCP + A2A.
- `docker-compose up` runs locally.

### Phase 1 — Core engine, single hardcoded tenant (months 1–3)

- All 12 agents implemented.
- LangGraph orchestrator wired.
- Inngest functions for async work.
- Postgres schema deployed.
- R2 + Langfuse + Nango wired.
- Master deck → python-pptx → personalized deck working.
- Gmail send + reply monitoring via Nango.
- **Test tenant: fictional "Hypothesis Software"** (generic B2B SaaS, keeps build industry-agnostic).
- Run 200 real-ish leads end-to-end; iterate output quality.
- App basics: Clerk auth, lead list, lead detail page (read-only).

**Exit criteria:** Output quality passes senior-salesperson "would send as-is" bar on 9/10 random samples.

### Phase 2 — Multi-tenant + onboarding (months 3–5)

- RLS enabled on all tables.
- Clerk multi-org wired.
- Vector namespaces per tenant.
- R2 prefix isolation enforced.
- Full conversational onboarding flow live.
- App: settings (brand voice, ICP, offerings, master deck, integrations, API keys, team).
- Internal admin app skeleton at `admin.<domain>`.
- **Onboard Elchai's 5 brands** as tenants.
- **Invite 3–5 friendly external tenants** of varied industries.
- No billing, no public signup.

### Phase 3 — Agent protocol layer (months 5–6)

- FastMCP server live at `mcp.<domain>`.
- A2A agent card at `a2a.<domain>`.
- Auth on both transports.
- TS + Python SDKs auto-generated.
- Mintlify docs live with quickstarts.

### Phase 4 — Production hardening + optional integrations (months 6–9)

- Telegram integration (opt-in per tenant) — BYO bot pattern.
- Full webhook system with retries.
- Per-tenant observability dashboard.
- Eval suite running nightly.
- Insights/analytics page in app.
- Load testing.
- DR + backup verification.

### Phase 4.5 — Cost + readiness data (parallel with Phase 4)

- Per-lead cost attribution fully wired.
- Per-tenant economics dashboard.
- Time-saved metrics.
- **End-of-month-9 decision:** does usage data support commercialization?

### Phase 5 — Commercialization (months 9–12, gated)

Only if go decision at end of Phase 4.5:

- Stripe + Lago billing wired.
- Public signup at `app.<domain>/sign-up`.
- Pricing page inside app (no separate marketing site).
- SOC 2 Type I prep.
- Status page at `status.<domain>`.
- Separate brand/SPV setup if pursuing.

---

## 14. Testing Strategy

- **Pyramid:** unit 70 / integration 20 / eval 8 / E2E 2.
- **Evals:** per-agent suite, JSONL dataset, scored on schema validity (100%), semantic checks, cost, latency. Run nightly + on prompt changes. Pushed to Langfuse.
- **Production probes:** every 5 minutes synthetic tenant calls each capability with known input. Fails → page on-call.

---

## 15. Deployment

- **Environments:** dev (local docker-compose), staging (Railway + Neon staging branch), production (Railway → AWS later).
- **CI/CD:** PR opened → lint, type-check, tests on changed packages. Merge to `main` → staging deploy. Manual promotion to production.
- **Migrations:** Alembic, reversible only, run in separate migration job blocking app rollout.

---

## 16. Security + Compliance

- TLS 1.3 only.
- API keys hashed with argon2id.
- OAuth tokens encrypted at rest (Nango).
- Neon PITR, 30-day retention.
- R2 versioning enabled.
- Admin actions logged to immutable audit table.
- GDPR: data export, deletion, consent tracking.
- CAN-SPAM: `List-Unsubscribe` header, suppression list honored.
- LinkedIn: never exceed ToS rate limits.

---

## 17. For Claude Code — How to Work in This Repository

When implementing any feature:

1. **Read `CLAUDE.md` first.** Never deviate from locked decisions without an ADR.
2. **Find the relevant agent or service.** Most work touches `/packages/agents/<agent>/` or `/apps/api/` or `/apps/app/`.
3. **Write the schema first.** Pydantic for backend, Zod for frontend.
4. **Write the test first.** Happy-path before implementation.
5. **Run `ruff` + `mypy` + tests locally before commit.**
6. **ADR for any architectural decision not covered here.** Goes in `/docs/adr/NNN-title.md`.
7. **Update `CLAUDE.md` when behavior changes.**

### Implementation checklist for any new endpoint

- [ ] Pydantic input + output schemas
- [ ] Agent method on relevant agent class
- [ ] REST route + adapter in `/apps/api/`
- [ ] MCP tool exposing same capability
- [ ] A2A skill exposing same capability
- [ ] Inngest function if async
- [ ] Unit tests covering happy + error paths
- [ ] Eval cases if LLM involved
- [ ] OpenAPI spec regenerated
- [ ] SDK regenerated
- [ ] Docs page on Mintlify
- [ ] Audit log entry emitted
- [ ] Tenant cost recorded

### Implementation checklist for any new app page

- [ ] Route under `/apps/app/app/`
- [ ] Server Component by default, Client only if interactive
- [ ] Data fetched via tRPC + TanStack Query
- [ ] Forms via React Hook Form + Zod
- [ ] Loading + error states
- [ ] Mobile-responsive
- [ ] Realtime updates via SSE where state changes
- [ ] Playwright E2E test for happy path

---

## 18. Open Decisions

- [ ] Final product name (replace "arcadia" throughout).
- [ ] Pricing model (decide at Phase 4.5).
- [ ] Founding engineering team shape.
- [ ] Brand identity (logo, colors, type).
- [ ] Legal entity (Elchai-owned vs separate SPV — recommend separate SPV).
- [ ] Initial geographic focus.
- [ ] Friendly external tenants for Phase 2 cohort.
- [ ] Marketing site decision (deferred until commercialization).

---

## 19. Glossary

- **Tenant** — a customer organization (`tenant_id` UUID, backed by Clerk org). Includes internal Elchai brands and external customers.
- **Lead** — a target contact at a target company a tenant wants to convert to a meeting.
- **Crew** — the 12 agents working together for one tenant.
- **Master deck** — tenant's brand-compliant pitch deck template that Deck Builder personalizes per lead.
- **Brand voice** — tenant's structured voice profile (tone, vocabulary, samples).
- **ICP** — Ideal Customer Profile.
- **Capability** — a discrete unit of work exposed over REST/MCP/A2A.
- **Onboarder** — agent that extracts brand voice, ICP, offering at signup.
- **Prospector** — agent that discovers leads matching an ICP.

---

*Last updated: 2026-05-15. Version 2. Owner: Flavio Elia. Contributors must update this file when behavior changes.*
