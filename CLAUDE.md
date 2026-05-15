# CLAUDE.md — Project Architecture & Build Specification (v4, 9/10 plan)

> **Working name:** `arcadia` (placeholder — replace throughout when final name is chosen).
> **One-line description:** A multi-tenant, agent-native B2B revenue platform. Tenant signs up, conversational onboarding extracts brand voice and ICP, agent crew prospects, researches, generates personalized email and bespoke deck/video, sends, handles replies, books meetings.
> **Mission framing:** Long-term mission is to make traditional sales teams unnecessary for most B2B companies. Phases 0–4 ship the foundation: outbound sales that replaces SDR work entirely.
> **Status:** Specification locked. No code written yet.
> **Operating mode:** Internal-only Phases 0–4. Commercialization gated on Phase 4.5 decision.
> **Language scope:** English only. Multi-language deferred to Phase 5+.

This document is the source of truth for Claude Code and any engineer working on this codebase. Every architectural decision below is **locked** unless this document is updated via ADR.

---

## 0. Quick Reference (Claude Code session context)

**Critical rules to never violate** (full list §12):

1. `tenant_id` in every query
2. Vector namespace per tenant
3. R2 keys start with `tenants/<tenant_id>/`
4. All LLM calls through `arcadia_core/llm/client.py`
5. No PII in logs
6. Idempotency keys on mutating POSTs
7. Long-running ops via Modal/Inngest, never bare `asyncio.create_task`
8. Prompts in `.md` files, versioned in Langfuse
9. Schema-first (Pydantic backend, Zod frontend)
10. One adapter per transport
11. Parameterized queries only
12. Webhook payloads HMAC-signed
13. Test-mode keys never touch production integrations
14. LLM outputs validated by Pydantic, retry-once-then-fail
15. Suppression list consulted before every send
16. No industry/region/language hardcoded
17. No marketing site in Phases 0–4
18. No host-specific code in business logic
19. Per-tenant cost caps enforced before every LLM call
20. Every agent output meets §17 quality bar

**When implementing:** read §6 for the relevant agent, §17 for output quality bar, §12 for rules.

---

## 1. Product Specification

### 1.1 What we are building

A B2B revenue platform exposing the same agent crew through four transports:

- **Web app** at `app.<domain>` — operator's home
- **REST API** — developer integration
- **MCP server** at `mcp.<domain>` — AI assistants install capabilities as native tools
- **A2A peer** at `a2a.<domain>` — other agent platforms delegate tasks

Optional human-channel integration (tenant-opt-in, Phase 4+): Telegram via BYO-bot pattern.

Industry-agnostic. Brand voice, ICP, offering extracted from each tenant during conversational onboarding.

### 1.2 The agent crew

**13 core agents + 2 on-demand sub-builders = 15 total.**

| Agent | Type | Responsibility |
|---|---|---|
| Onboarder | Core | Extract brand voice, ICP, offering from tenant URL |
| Prospector | Core | Discover leads matching tenant's ICP |
| Orchestrator | Core | Lead state machine; routes to next agent |
| Intake | Core | Normalize incoming leads |
| Researcher | Core | Deep research dossier on company + person |
| Strategist | Core | Map offering to prospect; define campaign angle |
| Copywriter | Core | Draft personalized email |
| Sentinel | Core | Compliance + brand + quality gate |
| Verifier | Core | Factual grounding against cited sources |
| Sender | Core | Send via tenant's connected mailbox |
| Reply Handler | Core | Classify replies, detect sub-intent, route |
| Closer | Core | Sequence follow-ups; book meetings |
| Materials Coordinator | Core | Orchestrate on-demand sub-builders |
| **Deck Builder** | Sub | Bespoke pitch deck from master template |
| **Video Builder** | Sub | HyperFrames personalized motion-graphics video |

### 1.3 On-demand materials pattern

**Always generated:** research dossier + personalized email.
**On request:** pitch deck, video.

Triggers: upfront (specified at lead creation), manual (dashboard click), auto-triggered (Reply Handler detects `interested_request_for_materials`).

### 1.4 What we are NOT building

- Email deliverability infrastructure (tenants connect Gmail/Outlook)
- CRM (we sync via Nango)
- Prospecting databases (we integrate Apollo, Clay)
- Calendar booking engine (we integrate Cal.com, Calendly, Google Calendar)
- AI inference (we call Anthropic + OpenAI fallback)
- Marketing website (app + docs only)
- Multi-language support (Phase 5+)
- Voice/phone agents
- AI avatar video (different product category)

### 1.5 Operating mode

Phases 0–4: internal-only. No public signup, no billing, no marketing. Friendly tenants invited in Phase 2+.
Phase 4.5: commercialization decision gate.
Phase 5: commercialization if supported by data.

Phase timelines depend on team — see §22 for honest reckoning.

---

## 2. Technology Stack

### 2.1 Locked (architectural)

| Layer | Choice |
|---|---|
| Backend | Python 3.12 + FastAPI |
| Frontend | TypeScript + Next.js 15 (App Router) |
| Agent SDK | Anthropic Claude Agent SDK |
| Multi-agent orchestration | LangGraph (`>=0.4`) |
| MCP server | FastMCP |
| A2A | `a2a-python` (Google) |
| DB | Postgres 16 + pgvector + RLS |
| Object storage | Cloudflare R2 |
| Auth | Clerk |
| Primary LLM | Anthropic Claude |

### 2.2 Phase 0 defaults (ADR-confirmable at Phase 1 exit)

| Layer | Default | Note |
|---|---|---|
| Burst compute | Modal | |
| Durable workflows | Inngest hosted | Inngest decides "what runs next"; Modal runs each agent |
| Event bus | Postgres LISTEN/NOTIFY | Promote to Redpanda at sustained >1000 events/min |
| OAuth integrations | Nango | DIY acceptable for 2–3 integrations in Phase 1 |
| LLM tracing | Langfuse self-hosted | |
| Errors | Sentry | |
| Logs | Better Stack | |
| Fallback LLM | OpenAI | Activated on Anthropic outage |
| Deck rendering | python-pptx + LibreOffice headless | |
| Video rendering | HyperFrames (Apache 2.0, free) | Confirmed working — Phase 2 standard |
| Billing (Phase 5+) | Stripe + Lago | |
| Docs site | Mintlify | |

### 2.3 Hosting

| Component | Phase 0–4 |
|---|---|
| Frontend | Vercel Pro |
| Backend | DigitalOcean Premium AMD Droplet (4 vCPU / 8 GB / 240 GB NVMe, $56/month) |
| Burst workers | Modal |
| Database | Neon Scale → Business |
| Object storage | Cloudflare R2 |
| Observability | Langfuse self-host on $24 DO droplet |

**No AWS.** Phase 5+ enterprise migration target is GCP if needed.

### 2.4 Tooling

- `uv` (Python), `pnpm` (Node), `turborepo`, `uv workspaces`
- Phase 0–1 lint: `ruff` defaults, `mypy` not strict, 50% coverage
- Phase 2+ lint: `ruff` strict, `mypy --strict`, 80% coverage
- Testing: `pytest` + `httpx`, `vitest`, `playwright`
- CI: GitHub Actions
- Secrets: Doppler or Infisical

---

## 3. System Architecture

### 3.1 High-level flow

```
TENANT OPERATOR ──┐
                  ├──► app.<domain> (Next.js, Vercel)
TEAMMATES ────────┘            │
                               ▼
DEVELOPER APPS ───► REST API ──┤
AI ASSISTANTS ────► MCP server ┼─► API Gateway on DigitalOcean droplet
OTHER AGENTS ─────► A2A peer ──┘  (auth, tenant, rate limit, idempotency, cost cap)
TELEGRAM (opt-in) ► webhook ──────►│
                                   ▼
                  Capability Service (FastAPI + FastMCP + a2a-python)
                  Same process, all transports
                                   │
                                   ▼
                          LangGraph workflows
                          (checkpointed to Postgres)
                                   │
                                   ▼
                          Modal functions
                          (autoscaling containers per agent)
                                   │
                                   ▼
                  Agents call: Anthropic, Postgres+pgvector, R2,
                  Nango, web search, Apollo, Clearbit
                                   │
                                   ▼
                          Events → Postgres LISTEN/NOTIFY
                          Webhooks delivered to tenants
                          SSE realtime to app
```

### 3.2 One agent, four transports

Each agent implemented once, exposed four ways. Never duplicate logic.

```python
# packages/agents/researcher/agent.py
class ResearcherAgent:
    async def research_company(self, domain: str, tenant_id: str) -> ResearchDossier: ...

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
    return A2AArtifact(content=(await researcher.research_company(
        task.input["domain"], task.context.tenant_id
    )).model_dump())

# Web app: tRPC → REST. No separate adapter.
```

**Terminology lock:** "Transport" = REST | MCP | A2A. "Surface" = web app | Telegram | future channel.

### 3.3 Library ownership matrix

| Concern | Owner |
|---|---|
| Workflow state across agents | LangGraph checkpoints |
| Single-agent conversation memory | Claude Agent SDK Memory |
| Durable source of truth | Postgres `agent_runs.state` |
| Ephemeral session state | Transport's session object |
| Tool definitions for LLMs inside agent | Claude Agent SDK tools |
| Tool definitions for external agents | FastMCP / A2A (same Python wrapped) |
| Cost attribution | `arcadia_core/llm/client.py` wrapper |
| Long-running execution durability | Modal function (orchestrated by Inngest) |

**Modal vs Inngest:** Inngest coordinates workflow ("what runs next"). Modal runs each agent (compute substrate). Inngest owns retries, scheduling, step durability. Modal owns containers, autoscaling, per-function timeout.

### 3.4 Sync vs async + cold-start

| Latency | Pattern | Examples |
|---|---|---|
| < 2s p95 | Sync | classify reply, voice extract |
| 2–30s p95 | Sync with timeout fallback | research quick, generate email |
| > 30s expected | Async with job_id | deck, video, deep research, campaign |

**Cold-start mitigation:** Modal `keep_warm=2` on hot paths.

**Sync timeout escape:** every sync endpoint has 28-second hard timeout. If exceeded, server upgrades to async, returns `{job_id, status: "running", original_request: true}`. Prevents proxy cutoffs from breaking UX.

### 3.5 Embedding pipeline

Dedicated `EmbeddingService` (in `packages/core/embeddings/`) called from agents producing embeddable content.

- Embedded by Researcher: company facts, persona briefs, sources
- Embedded by Onboarder: brand voice samples, offerings, ICP descriptions
- Embedded by Reply Handler: classified reply contents for memory graph

Model: `voyage-3` (1536 dims). Triggered synchronously inside runs. Re-embedding by content hash.

Namespace: `tenant_<uuid>_<source_type>`.

### 3.6 Failure recovery patterns

Every agent has three failure modes in its §6 spec:

- **Recoverable** — automatic retry (Inngest exponential backoff, max 3)
- **Degraded** — fall back to lower-tier output with explicit confidence flag
- **Terminal** — escalate to human review, lead state → `needs_human`, tenant notified

LLM provider outages handled at wrapper layer: Anthropic primary, OpenAI fallback automatic on 5xx or sustained timeout. Tenant notified if outage > 5 min.

---

## 4. Multi-Tenant Isolation (NEVER VIOLATE)

Six layers. Each enforces tenant_id independently.

### 4.1 Database — Postgres RLS

```sql
CREATE POLICY tenant_isolation ON contacts
    FOR ALL USING (tenant_id = current_setting('app.current_tenant')::uuid);
ALTER TABLE contacts ENABLE ROW LEVEL SECURITY;
```

Gateway sets `SET app.current_tenant` per session. Cross-tenant admin queries use separate audited role.

### 4.2 Vector embeddings — namespace per tenant

Every similarity query includes `WHERE namespace = ?`. Cross-tenant search forbidden.

### 4.3 Object storage — workspace per tenant

```
r2://arcadia-artifacts/tenants/<tenant_id>/
    decks/<lead_id>/...
    videos/<lead_id>/...
    brand/...
    research/<company_id>/...
```

Signed URLs, expiry < 1 hour.

### 4.4 Agent context + memory

Every Agent SDK Memory, LangGraph state, prompt cache key includes tenant_id.

### 4.5 Cost attribution

Every LLM call logged with `{tenant_id, agent, model, input_tokens, output_tokens, cost_usd, request_id}`. Direct SDK calls outside the wrapper are forbidden.

### 4.6 Per-tenant cost caps

`tenant_quotas` table. Wrapper checks budget *before* every call:

```python
async def call_llm(tenant_id, agent, model, messages):
    quota = await get_tenant_quota(tenant_id)
    if quota.spent_today_usd >= quota.daily_limit_usd:
        raise QuotaExceeded("daily", tenant_id)
    if quota.spent_month_usd >= quota.monthly_limit_usd:
        raise QuotaExceeded("monthly", tenant_id)
```

Defaults Phase 0–4: $50/day, $1000/month per tenant. Soft warning at 80% (webhook + dashboard). Hard stop at 100% (in-flight runs complete). Phase 5+ set by billing plan.

---

## 5. Data Model

Alembic migrations in `/apps/api/migrations/`. Every table has RLS with `tenant_isolation`.

### 5.1 Tenancy + auth

```sql
CREATE TABLE organizations (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    clerk_org_id TEXT UNIQUE NOT NULL,
    name TEXT NOT NULL,
    plan TEXT NOT NULL DEFAULT 'internal',
    onboarding_state TEXT NOT NULL DEFAULT 'pending',
    settings JSONB DEFAULT '{}'::jsonb,
    created_at TIMESTAMPTZ DEFAULT NOW()
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

CREATE TABLE tenant_quotas (
    tenant_id UUID PRIMARY KEY REFERENCES organizations(id),
    daily_limit_usd NUMERIC(10,2) NOT NULL DEFAULT 50.00,
    monthly_limit_usd NUMERIC(10,2) NOT NULL DEFAULT 1000.00,
    spent_today_usd NUMERIC(10,2) NOT NULL DEFAULT 0,
    spent_month_usd NUMERIC(10,2) NOT NULL DEFAULT 0,
    last_reset_day DATE NOT NULL DEFAULT CURRENT_DATE,
    last_reset_month DATE NOT NULL DEFAULT date_trunc('month', CURRENT_DATE)::date,
    updated_at TIMESTAMPTZ DEFAULT NOW()
);
```

### 5.2 Tenant configuration

```sql
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
    master_video_template_r2_key TEXT,
    created_at TIMESTAMPTZ DEFAULT NOW()
);
```

### 5.3 Contact graph + memory

```sql
CREATE TABLE companies (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES organizations(id),
    domain TEXT NOT NULL,
    name TEXT,
    enriched_data JSONB DEFAULT '{}'::jsonb,
    research_dossier_r2_key TEXT,
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

CREATE TABLE prospect_facts (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES organizations(id),
    contact_id UUID NOT NULL REFERENCES contacts(id),
    fact_type TEXT NOT NULL,
    content TEXT NOT NULL,
    confidence REAL NOT NULL DEFAULT 0.8,
    source_message_id UUID,
    extracted_at TIMESTAMPTZ DEFAULT NOW()
);
CREATE INDEX idx_prospect_facts_contact ON prospect_facts(tenant_id, contact_id);

CREATE TABLE conversation_memory (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES organizations(id),
    contact_id UUID NOT NULL REFERENCES contacts(id),
    summary TEXT NOT NULL,
    last_updated TIMESTAMPTZ DEFAULT NOW(),
    interaction_count INT NOT NULL DEFAULT 0
);
```

### 5.4 Leads + pipeline + materials

```sql
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
    materials_requested TEXT[] DEFAULT '{}',
    last_event_at TIMESTAMPTZ DEFAULT NOW(),
    created_at TIMESTAMPTZ DEFAULT NOW()
);
CREATE INDEX idx_leads_tenant_state ON leads(tenant_id, state);

CREATE TABLE sequence_states (
    lead_id UUID PRIMARY KEY REFERENCES leads(id),
    tenant_id UUID NOT NULL REFERENCES organizations(id),
    current_step INT NOT NULL DEFAULT 0,
    branch TEXT NOT NULL DEFAULT 'default',
    next_action_at TIMESTAMPTZ,
    last_signal JSONB,
    updated_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE material_requests (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES organizations(id),
    lead_id UUID NOT NULL REFERENCES leads(id),
    material_type TEXT NOT NULL CHECK (material_type IN ('deck', 'video', 'one_pager', 'roi_calculator')),
    trigger_source TEXT NOT NULL CHECK (trigger_source IN ('upfront', 'manual', 'auto_reply', 'dashboard')),
    status TEXT NOT NULL DEFAULT 'queued',
    artifact_r2_key TEXT,
    artifact_url TEXT,
    cost_usd NUMERIC(10, 6),
    started_at TIMESTAMPTZ DEFAULT NOW(),
    completed_at TIMESTAMPTZ,
    error JSONB
);
```

### 5.5 Email, agent runs, cost, outcomes

```sql
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

CREATE TABLE agent_runs (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES organizations(id),
    agent_name TEXT NOT NULL,
    lead_id UUID REFERENCES leads(id),
    modal_call_id TEXT,
    inngest_run_id TEXT,
    state JSONB NOT NULL,
    started_at TIMESTAMPTZ DEFAULT NOW(),
    completed_at TIMESTAMPTZ,
    status TEXT NOT NULL DEFAULT 'running',
    failure_mode TEXT,
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

CREATE TABLE outcomes (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES organizations(id),
    lead_id UUID REFERENCES leads(id),
    agent_run_id UUID REFERENCES agent_runs(id),
    prompt_version TEXT NOT NULL,
    outcome_type TEXT NOT NULL,
    days_to_outcome INT,
    recorded_at TIMESTAMPTZ DEFAULT NOW()
);
CREATE INDEX idx_outcomes_prompt ON outcomes(prompt_version, outcome_type);
```

### 5.6 Events, integrations, suppression, embeddings, etc.

```sql
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

CREATE TABLE integrations (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES organizations(id),
    provider TEXT NOT NULL,
    nango_connection_id TEXT NOT NULL,
    status TEXT NOT NULL DEFAULT 'active',
    metadata JSONB DEFAULT '{}'::jsonb,
    UNIQUE (tenant_id, provider)
);

CREATE TABLE suppression_list (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES organizations(id),
    email TEXT,
    domain TEXT,
    reason TEXT NOT NULL,
    CHECK (email IS NOT NULL OR domain IS NOT NULL)
);

CREATE TABLE embeddings (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES organizations(id),
    namespace TEXT NOT NULL,
    source_type TEXT NOT NULL,
    source_id UUID,
    source_hash TEXT NOT NULL,
    content TEXT NOT NULL,
    embedding vector(1536) NOT NULL,
    metadata JSONB DEFAULT '{}'::jsonb,
    created_at TIMESTAMPTZ DEFAULT NOW()
);
CREATE INDEX idx_embeddings_namespace ON embeddings(tenant_id, namespace);

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

CREATE TABLE idempotency_keys (
    key TEXT PRIMARY KEY,
    tenant_id UUID NOT NULL REFERENCES organizations(id),
    request_hash TEXT NOT NULL,
    response JSONB,
    status_code INT,
    expires_at TIMESTAMPTZ NOT NULL
);

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

CREATE TABLE eval_runs (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    agent_name TEXT NOT NULL,
    prompt_version TEXT NOT NULL,
    suite_name TEXT NOT NULL,
    cases_total INT NOT NULL,
    cases_passed INT NOT NULL,
    avg_score REAL,
    metadata JSONB DEFAULT '{}'::jsonb,
    run_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE monitored_signals (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES organizations(id),
    company_id UUID NOT NULL REFERENCES companies(id),
    signal_type TEXT NOT NULL,
    signal_payload JSONB NOT NULL,
    detected_at TIMESTAMPTZ DEFAULT NOW(),
    triggered_action_id UUID
);
```

---

## 6. Agent Specifications

Each agent in `/packages/agents/<name>/`. Files: `agent.py`, `prompts/` (.md), `tools.py`, `schemas.py`, `tests/`, `evals/`.

Every spec includes: purpose, input, output, tools, model, failure modes (recoverable/degraded/terminal), quality bar reference.

### 6.1 Onboarder

**Purpose:** Extract brand voice, ICP, offering from tenant URL at signup.
**Input:** `{ company_url, optional_clarifications? }`.
**Output:** Draft `BrandVoice`, `IcpProfile`, `Offering`.
**Tools:** `web_fetch`, `web_search`, `parse_pptx`.
**Model:** Sonnet.
**Failure modes:**
- Recoverable: site unreachable → retry 3x with backoff
- Degraded: insufficient signal → `confidence: low`, request manual input
- Terminal: site doesn't exist → escalate
**Quality bar:** §17.4.

### 6.2 Prospector

**Purpose:** Find leads matching tenant's ICP.
**Input:** `{ icp_profile_id, volume, exclusions? }`.
**Output:** Scored leads with sources.
**Tools:** Apollo, LinkedIn Sales Nav (Nango), Crunchbase, web search.
**Model:** Sonnet for query gen; deterministic dedup.
**Failure modes:**
- Recoverable: rate limit → backoff
- Degraded: partial volume → return with `partial: true`
- Terminal: ICP unmatchable → escalate to tenant

### 6.3 Orchestrator

**Purpose:** Lead state machine. Routes each lead.
**Implementation:** LangGraph + Postgres checkpointing.

```python
graph = StateGraph(LeadState)
graph.add_node("intake", intake_node)
graph.add_node("research", research_node)
graph.add_node("strategize", strategize_node)
graph.add_node("copywrite", copywriter_node)
graph.add_node("sentinel", sentinel_node)
graph.add_node("verifier", verifier_node)
graph.add_node("send", sender_node)
graph.add_node("wait_for_reply", wait_node)
graph.add_node("handle_reply", reply_handler_node)
graph.add_node("close", closer_node)
graph.add_node("materials_coordinator", materials_node)
graph.add_node("escalate_human", human_review_node)
graph.add_conditional_edges("sentinel", route_sentinel)
graph.add_conditional_edges("verifier", route_verifier)
graph.add_conditional_edges("handle_reply", route_reply)
graph.add_conditional_edges("close", route_close)
```

### 6.4 Intake

Validates, dedups, enriches (Apollo if connected), fit-screens. Below threshold → `disqualified`.

### 6.5 Researcher

**Purpose:** Structured research dossier. **Output meets §17.1 quality bar.**
**Input:** `{ domain, person_email?, depth: "quick"|"standard"|"deep" }`.
**Output:** `ResearchDossier`.
**Tools:** `web_search`, `web_fetch`, `fetch_linkedin_company`, `query_company_database`, `fetch_news`, `fetch_funding`, `fetch_job_postings`, `fetch_tech_stack`.
**Model:** Sonnet (quick/standard), Opus (deep).
**Failure modes:**
- Recoverable: tool timeout → retry; transient errors → backoff
- Degraded: insufficient info → `confidence: low`, `requires_human_input: true`; never hallucinate
- Terminal: domain doesn't resolve → mark `disqualified`

### 6.6 Strategist

**Input:** `ResearchDossier`, `Offering`, `IcpProfile`, `BrandVoice`, `prospect_facts`.
**Output:** `CampaignStrategy`.
**Model:** Sonnet.
**Failure modes:**
- Degraded: weak dossier → best-effort with `confidence: medium`
- Terminal: offering doesn't fit → recommend disqualification

### 6.7 Copywriter

**Input:** `CampaignStrategy`, `ResearchDossier`, `deck_url?`, `BrandVoice`, `previous_messages?`, `prospect_facts`.
**Output:** `{ subject, body_text, body_html, references_used, sequence_branch }`. **§17.2 quality bar.**
**Constraint:** Every email MUST reference ≥2 specific facts from `ResearchDossier.sources`.
**Model:** Sonnet.
**Failure modes:**
- Recoverable: Pydantic validation fail → retry once with feedback
- Degraded: Verifier rejects → rewrite up to 2x
- Terminal: 3 rejections → `awaiting_human_review`

### 6.8 Sentinel

**Checks in order:**
1. Suppression list
2. Consent state (GDPR, CCPA)
3. Frequency cap per contact/domain
4. Brand compliance
5. Quality threshold (judge-LLM against §17.2)
6. Prohibited content

**Models:** Haiku for checks, Sonnet for rejection reasons.
**Never bypassable.**

### 6.9 Verifier

**Purpose:** Factual grounding. Every claim checked against sources.
**Implementation:** Parallel Haiku verification per claim.
**Cost:** ~$0.05/email.
**Failure modes:**
- Recoverable: API error → retry
- Degraded: borderline → flag for human, don't auto-fail
- Terminal: 3 grounding failures → escalate

### 6.10 Sender

Sends via Nango → Gmail/Outlook. Adds `List-Unsubscribe` (RFC 8058). Captures message-id + thread-id.
**Failure modes:**
- Recoverable: provider 5xx → retry (1m/5m/15m)
- Degraded: quota exceeded → queue next day
- Terminal: account suspended → reconnect required

### 6.11 Reply Handler

**Classifications:** `interested`, `interested_request_for_materials`, `interested_book_now`, `objection`, `question`, `out_of_office`, `not_interested`, `unsubscribe`, `wrong_person`, `auto_reply`, `other`.

**Routing:** see Section 6.11 v3 spec — unchanged. Side effect: every classified reply triggers `prospect_facts` extraction → memory graph update.

**Model:** Haiku. Confidence < 0.7 → escalate.

### 6.12 Closer

Multi-step sequencing with signal-aware branching:
- Lead opened 4x but no reply → `engaged_no_reply` branch
- Lead clicked deck → `clicked_deck` branch (deck-aware response)
- OOO → snooze precisely
- Company signal detected → re-sequence with new angle

Max 4 back-and-forth before mandatory human escalation. Books via connected calendar.
**Model:** Sonnet.

### 6.13 Materials Coordinator

Spawns Deck Builder + Video Builder (+ future) in parallel on materials request. Partial completion acceptable.

### 6.14 Deck Builder (on-demand)

**Output meets §17.3 quality bar.**
**Implementation:** Load master deck → parse placeholders → generate per-slide content → swap logo (Clearbit) → save PPTX → render PDF (LibreOffice headless) → upload R2.
**Cost target:** ~$0.50/deck. Ceiling: $1.
**Failure modes:** rendering retry once; unparseable elements → skip with warning; corrupted master → require re-upload.

### 6.15 Video Builder (on-demand, Phase 2 commitment)

**Implementation:** Load HyperFrames HTML template → generate per-scene content → Kokoro TTS (local, free) → substitute → `hyperframes render` in Modal → extract GIF thumbnail → upload R2.
**Cost target:** ~$0.15/video (no licensing fees — Apache 2.0).

---

## 7. API Surface

### 7.1 Versioning + auth

All endpoints under `/v1/`. Breaking changes → `/v2/`. 12-month deprecation support.

`Authorization: Bearer <api_key>`. Keys prefixed `arc_test_` or `arc_live_`.

### 7.2 Endpoints

**Atomic (sync):** `POST /v1/research/company`, `POST /v1/research/person`, `POST /v1/generate/email`, `POST /v1/classify/reply`, `POST /v1/voice/extract`.

**Generation (async):** `POST /v1/generate/deck`, `POST /v1/generate/video`, `POST /v1/sequence/generate`.

**Workflows (async):** `POST /v1/campaign/run`, `POST /v1/inbound/qualify`.

**Pipeline:** `POST/GET/DELETE /v1/leads`, `POST /v1/leads/{id}/replay`, `POST/GET /v1/leads/{id}/materials`.

**Jobs:** `GET /v1/jobs/{id}`.

**Webhooks:** `GET/POST/DELETE /v1/webhooks`, `POST /v1/webhooks/{id}/test`.

**Tenant config:** brand-voice, offerings, master-deck, master-video-template, icp-profiles, integrations, auto-trigger-policy, quotas, usage.

### 7.3 Idempotency + rate limits

Every POST accepts `Idempotency-Key: <uuid>`. 24h cache.
Rate limits: 100 req/min sync, 10 req/min async-job-starts (per tenant, configurable).

### 7.4 MCP + A2A

MCP at `https://mcp.<domain>/v1`. A2A card at `https://a2a.<domain>/.well-known/agent.json`.

---

## 8. Frontend Architecture

### 8.1 Surfaces

| Surface | URL | Audience |
|---|---|---|
| App | `app.<domain>` | Tenant operators |
| Docs | `docs.<domain>` | Developers (Mintlify) |
| Admin (Phase 2+) | `admin.<domain>` | Internal team |

**No marketing site.** Signup inside app.

### 8.2 App routes

```
/sign-in /sign-up /onboarding /
/leads/[id] /pipeline /campaigns /insights
/settings/{brand-voice, icp, offerings, master-deck, master-video-template,
          integrations, api-keys, team, telegram, quotas, auto-trigger-policy,
          billing}
```

### 8.3 Onboarding flow

1. Company URL → Onboarder extracts (15–30s)
2. Confirm extracted profile (editable)
3. Upload master deck (skippable)
4. Connect email + calendar (Nango)
5. Preview moment — type any company, full pipeline runs. **SLO: under 2 minutes target; 90s aspirational, not contract.**
6. Done. API key + MCP + A2A URLs shown.

### 8.4 Lead detail page

Summary header → research dossier (collapsible) → strategy (collapsible) → **Materials panel** (✓ Email sent, ○ Deck/Video/One-pager [Generate]) → email thread with classification badges → **inspectable agent run trace** (every step, model, cost, timing, prompt version) → actions panel.

### 8.5 API keys page

Test + live keys, MCP + A2A URLs visible. New key shown once in modal.

### 8.6 Realtime

SSE from FastAPI. No polling.

### 8.7 Admin app (Phase 2+)

Separate domain. View tenants, agent runs, suspend abusive tenants. Audited RLS bypass.

---

## 9. Event Bus

### 9.1 Phase 0–2: Postgres LISTEN/NOTIFY

```sql
NOTIFY agent_events, '{"id":"evt_...","tenant_id":"...","type":"...",...}';
```

### 9.2 Promotion to Redpanda — gated on:

- Sustained >1000 events/min, OR
- More than 3 distinct consumer services, OR
- Replay requirements LISTEN/NOTIFY can't satisfy, OR
- Tenant requires dedicated topic for compliance

Migration = 1 ADR + new infrastructure + consumer library swap. ~1 week.

### 9.3 Event catalog

**Lead lifecycle:** received, researched, strategized, email_drafted, email_approved, email_rejected, email_sent, replied, qualified, booked, lost, unsubscribed.
**Materials:** requested, generated, failed.
**Agent:** started, completed, failed, escalated.
**Tenant:** created, onboarding_completed, integration_connected.
**Outcomes:** recorded (feeds prompt evolution).
**Signals:** detected, triggered_action.
**Quota:** warning_80pct, exceeded, reset.

### 9.4 Webhook delivery

HMAC-signed. Retry 5x exponential. Failed → degraded.

---

## 10. Project Structure

```
arcadia/
├── CLAUDE.md
├── docs/{adr,runbooks,domain-glossary.md}
├── apps/{api,app,docs,admin,workers}
├── packages/
│   ├── agents/src/arcadia_agents/{onboarder,prospector,orchestrator,intake,
│   │     researcher,strategist,copywriter,sentinel,verifier,sender,
│   │     reply_handler,closer,materials_coordinator,deck_builder,video_builder,
│   │     common,tools}
│   ├── core/src/arcadia_core/{db,events,nango,llm,embeddings,quotas,
│   │     observability,schemas}
│   ├── ui sdk-python sdk-typescript
├── infra/{terraform,docker,deploy}
├── scripts/{seed-dev-tenant.py,run-eval.py,generate-openapi.py,golden-dataset/}
├── .github/workflows/
└── docker-compose.yml
```

---

## 11. Coding Conventions

**Python 3.12:** async I/O, Pydantic v2, SQLAlchemy 2.x async, `structlog` only.
**TypeScript:** strict + `noUncheckedIndexedAccess`, no `any`, Server Components default, Zod at boundaries.
**Prompts:** `.md` files in `prompts/`, versioned in Langfuse.
**Tests:** unit 70 / integration 20 / eval 8 / E2E 2.
**Logs:** every log includes `tenant_id`, `trace_id`, `agent_name`.
**Errors:** domain-typed; map to HTTP at boundary.

---

## 12. Critical Rules (CI-enforced)

1. `tenant_id` in every query
2. Vector namespace per tenant
3. R2 keys tenant-prefixed
4. All LLM via `arcadia_core/llm/client.py`
5. No PII in logs
6. Idempotency keys on mutating POSTs
7. Long-running via Modal/Inngest
8. Prompts in `.md`, versioned in Langfuse
9. Schema-first
10. One adapter per transport
11. Parameterized queries only
12. Webhooks HMAC-signed
13. Test keys never touch production
14. Pydantic validation, retry-once-then-fail
15. Suppression list before every send
16. **No industry/language/region hardcoded.** CI lint scans business logic
17. No marketing site Phases 0–4
18. No host-specific code in business logic
19. **Per-tenant cost caps enforced before every LLM call**
20. **Every agent output meets §17 quality bar** — measured, not assumed

---

## 13. Build Sequence

### Phase 0 — Skeleton (1–2 weeks, 1–2 people)

- Monorepo bootstrap
- One agent (Researcher) end-to-end
- REST + MCP + A2A
- `docker-compose up` works
- Test verifies all transports return identical output
- **§17.4 golden dataset creation begins in parallel**

### Phase 1 — Foundation, single hardcoded tenant (3–5 months, depends on team — §22)

**Always:**
- Postgres schema (RLS optional in single-tenant phase)
- R2 + Langfuse + Anthropic + LLM wrapper with cost capping
- Gmail send + reply monitoring
- Master deck → personalized deck working
- Eval suite weekly against golden dataset
- App: Clerk auth, lead list, lead detail page, inspectable run trace
- Test tenant: fictional "Hypothesis Software"
- **Verifier operational from week 1**
- **Outcomes table populated; prompt-evolution scaffolded**

**Founder mode (1–2 engineers):**
- 6 agents: Onboarder, Researcher, Strategist, Copywriter, Sentinel, Sender + Verifier
- Reply Handler, Closer, Prospector, Materials Coordinator → Phase 2
- REST only; MCP + A2A → Phase 3
- 50–100 leads through pipeline
- **Exit criteria:** golden-dataset pass rate >80% on Researcher + Copywriter

**Funded mode (3–4 engineers):**
- All 13 + 2 sub-builders
- REST + MCP + A2A from week 4
- LangGraph orchestrator with all edges
- Modal wrapping every agent
- 200+ leads
- **Exit criteria:** >85% across all agents; output passes "senior salesperson would send 9/10" review

### Phase 2 — Multi-tenant + onboarding (2–3 months, requires 3+ engineers minimum)

- RLS enabled
- Clerk multi-org
- Vector namespaces enforced
- R2 prefix isolation
- **Per-tenant cost caps live**
- Conversational onboarding flow
- App settings (brand-voice, ICP, offerings, master deck, integrations, API keys, team, quotas)
- Admin app skeleton
- 5–10 friendly invited tenants of varied industries
- **Video Builder operational** (HyperFrames)
- **Reply-rate-driven prompt evolution fully active**
- **Materials Coordinator + on-demand API**

Founder mode that proceeded alone in Phase 1 hits a wall here. §22 decision.

### Phase 3 — Agent protocol + signals (1–2 months)

- FastMCP server live
- A2A agent card
- Stainless-generated SDKs
- Mintlify docs live
- **Buying-signal monitoring** subsystem
- **Conversation Memory Graph populated and used**

### Phase 4 — Production hardening (3 months)

- Telegram opt-in integration
- Full webhook retries
- Per-tenant dashboards
- Eval suite nightly
- Insights/analytics page
- Load testing → fixes
- DR + backup verified
- **All §21 runbooks documented and tested**

### Phase 4.5 — Commercialization-readiness data (parallel with Phase 4)

Per-lead cost attribution wired. Per-tenant economics dashboard. Time-saved metrics. **Decision gate.**

### Phase 5 — Commercialization (3–6 months, gated)

- Stripe + Lago billing
- Public signup
- Pricing inside app
- SOC 2 Type I prep
- Status page
- Separate brand/SPV if pursuing
- **Pricing:** per-lead ($12–15) with monthly minimum, OR hybrid subscription + usage. Outcome-based ($300/booked-meeting) premium.

---

## 14. Testing Strategy

- Pyramid: unit 70 / integration 20 / eval 8 / E2E 2
- Evals: per-agent JSONL from §17.4. Scored on schema (100%), §17 criteria (judge-LLM), cost, latency. Weekly Phase 1, nightly Phase 2+.
- Production probes: every 5 min synthetic tenant calls each capability. Fails → page on-call.
- **CI lints (rule #16):** scan for hardcoded industry/language/region strings.

---

## 15. Deployment + Operations

### 15.1 Environments

- **dev** — `docker-compose up` (one command)
- **staging** — DO staging droplet + Neon staging branch
- **production** — DO production droplet + Neon main + Modal production

### 15.2 Local dev one-command setup

```bash
git clone <repo>
cd arcadia
./scripts/dev-setup.sh    # installs uv+pnpm, creates .env, seeds dev tenant
docker-compose up
```

Must work on macOS + Ubuntu. < 10 minutes from clone to local instance.

### 15.3 CI/CD

PR → lint, type-check, tests on changed packages. Merge to `main` → staging auto-deploy. Manual production promotion after smoke tests. Modal via `modal deploy` in CI. Migrations in separate blocking job.

### 15.4 Migrations + rollback

Alembic, reversible only. Code: previous Docker image one command away; `vercel rollback` for frontend. DB: every migration reversible. Modal: previous version one command away. **DO Droplet snapshots:** daily automated, one-click restore.

### 15.5 Cost estimates

| Phase | Monthly infra |
|---|---|
| Phase 0–1 (dev) | ~$200 |
| Phase 2–3 (5–10 tenants) | ~$500–700 |
| Phase 4 | ~$1,000–1,800 |
| Phase 5 (50 tenants) | ~$4,500–6,500 (5–10% of MRR) |

### 15.6 Backup strategy

- **Postgres:** Neon PITR, 30-day retention
- **R2:** versioning on production bucket; weekly cross-region snapshot
- **Langfuse:** self-hosted Postgres PITR
- **Configuration/secrets:** Doppler/Infisical backup; weekly encrypted export to R2
- **Code:** GitHub canonical; weekly mirror to second Git host

---

## 16. Security + Compliance

- TLS 1.3 only
- API keys hashed argon2id
- OAuth tokens encrypted at rest
- Neon PITR 30-day
- R2 versioning
- Immutable audit table (append-only, separate role)
- GDPR: data export + deletion endpoints, consent state tracking
- CAN-SPAM: `List-Unsubscribe` every outbound, suppression honored
- LinkedIn: never exceed ToS rate limits
- **Secrets rotation:** API keys quarterly; OAuth via Nango; webhook secrets on-demand; DB passwords semi-annually
- **Abuse detection:** flags on (a) 5x normal LLM spend in a day, (b) >50% suppressed contacts, (c) 10+ leads/min sustained → automatic pause + investigation
- **Audit log:** admin actions, key revocations, tenant suspensions, data exports, PII access

---

## 17. Output Quality Bar

The single most important section. Without measurable bars and a golden dataset, the platform is mediocre regardless of architecture.

### 17.1 Research dossier bar

What a senior human SDR produces given 30 minutes per account.

Required:
- **Company facts (6–10):** name, industry, size estimate (with source), stage, HQ, geography, products, customers, revenue range, growth trajectory
- **Recent signals (≥3, dated, sourced):** funding, exec changes, product launches, hiring patterns, market expansion, public statements, regulatory. Each has `relevance_to_offering`
- **Likely pain points (2–3):** tied to evidence, not generic
- **Decision-makers (1–3):** name, title, tenure_months, prior_role, LinkedIn URL, fit_score, rationale
- **Tech stack indicators** when detectable
- **Competitive position:** what they likely use that competes; what they say about pain
- **Fit assessment:** explicit qualified/disqualified + reasoning
- **Timing signals:** why-now-vs-later with evidence
- **Sources:** every fact tied to URL with `fetched_at`
- **Confidence:** high/medium/low + rationale

**Fail conditions:** generic content, unsourced claims, missing decision-maker when reasonable, no timing assessment, hallucinated facts.

### 17.2 Personalized email bar

What a senior human seller writes given the dossier.

Hard requirements:
- **Length:** 75–150 words
- **Specificity:** ≥2 facts from sources woven naturally
- **Subject:** under 8 words, specific
- **Single ask:** one clear next step
- **Voice match:** consistent with `BrandVoice`
- **CTA strength:** specific, time-bounded
- **No spam triggers**

**AI-tell ban list:**
- "I hope this email finds you well"
- "I wanted to reach out"
- "I'd love to explore"
- "I came across your"
- "Quick question"
- Em-dash sentences for emphasis
- Rule-of-three patterns
- Hedging ("perhaps", "might", "could potentially")
- "Don't hesitate to"
- "Looking forward to hearing from you"

**Industry-standard reply rate target:** 4–8% tier-1, 2–4% tier-2. Below 2% sustained = not industry standard.

### 17.3 Pitch deck bar

What a senior salesperson builds given 2 hours and the master deck.

Required:
- **First slide title** references prospect's specific situation
- **Problem statement** in prospect's actual language, references specific pain
- **Solution mapping** to prospect's situation
- **Case study slide** picks most-similar reference (industry/size/use-case match)
- **ROI slide** uses real numbers (headcount, growth, savings)
- **Visual consistency** with master deck preserved
- **Length** matches master deck

**Fail conditions:** master deck with name swapped on slide 1, generic pain points, random case study, percentages without specific numbers.

### 17.4 Golden dataset discipline (Phase 0 deliverable)

**Before any agent code is written:** founder + senior contractor produce 50–100 hand-crafted prospect examples.

Each example:
- Real (or realistic) target company
- Ideal research dossier (§17.1)
- Ideal email (§17.2)
- Ideal personalized deck content (§17.3, slide-by-slide)

Stored in `/scripts/golden-dataset/` as JSONL with structured fields.

Used as:
- Eval suite ground truth
- Few-shot examples in prompts
- Training data for judge-LLM rubric
- Calibration for human review

**Cost:** ~$5–10k contractor + 80 hours founder. Cheaper than any other quality investment.

**Without this: mediocre. With it: credibly best-in-industry.**

### 17.5 Eval discipline

- Schema validation: 100% required
- Rubric scoring: judge-LLM (Sonnet) compares to golden examples
- Human spot-check: founder reviews 10 random outputs weekly
- Scores in Langfuse per prompt version; regressions block updates
- Phase 1: weekly. Phase 2+: nightly + weekly human.

---

## 18. Best-in-Industry Commitments

### 18.1 Committed (Phase 1–3)

1. **Reply-rate-driven prompt evolution loop**
2. **Verifier Agent** (factual grounding)
3. **Buying-signal real-time monitoring** (Phase 3)
4. **Conversation Memory Graph**

(HyperFrames Video Builder is now standard capability, not "best in industry" item.)

### 18.2 Deferred (Phase 4+ candidates, ADR-required)

Multi-model orchestration, pre-meeting briefing, per-tenant voice fine-tuning, multi-channel orchestration, auto-skill library, multi-language native.

---

## 19. Competitive Positioning

### 19.1 Named competitors

| Competitor | Position | Strength | Weakness |
|---|---|---|---|
| 11x.ai | Direct, $74M+ raised | Brand, GTM | Mediocre output, no MCP/A2A |
| Artisan AI | Direct, $1B+ valuation | Marketing | Templated personalization |
| AiSDR | Direct | Low price | Output quality |
| Apollo (their AI) | Incumbent + AI | Massive data | AI bolt-on |
| HubSpot Breeze | Platform incumbent | CRM gravity | Generic AI |
| Salesforce Agentforce | Enterprise platform | Reach | Heavy, slow, not for SMB |
| Outreach/Salesloft AI | Sequencer incumbents | Embedded in workflow | AI bolt-on |

### 19.2 Our differentiation

- **Bespoke deck + video per account**
- **Agent-native: MCP + A2A from day one**
- **Verifier-grounded output** (no hallucinated facts)
- **On-demand materials economics**
- **Eval-driven quality** (most don't measure systematically)

### 19.3 Where we DON'T compete

AI avatar/talking-head video. Voice/phone outbound. Full CRM replacement. Marketing automation (Marketo, Pardot).

### 19.4 GTM thesis (Phase 5)

Sell to founder-led B2B companies $1M–$50M ARR. Buyer = founder or first sales hire. Wedge: "Replace your first 5 SDR hires with a platform better than them." Distribution: founder-to-founder referrals + agentic ecosystem (Claude/ChatGPT users discover via MCP installation) + content (case studies showing cost savings).

### 19.5 Data moat

Three compounding moats per tenant per month:
- **Conversation memory** — per-tenant prospect interaction history
- **Outcome data** — reply rates, booking rates by prompt version
- **Brand voice refinement** — voice profile improves with usage

After 6 months: switching cost significant.

---

## 20. Customer Lifecycle (Phase 5 commitment)

### 20.1 Time-to-value SLO

- Sign-up → onboarding complete: < 10 minutes
- Onboarding → first lead sent: < 1 hour
- First lead → first reply: median 3 days
- First reply → first booked meeting: median 7 days
- **Total TTV (sign-up to first booked): under 14 days**

Single most important commercial metric.

### 20.2 Activation criteria

- Onboarding complete
- ≥1 integration connected
- ≥10 leads processed
- ≥1 reply received

Not activated in 14 days → outreach flagged.

### 20.3 Retention mechanics

- Memory accumulates → switching cost
- Per-tenant prompt optimization → quality improves with usage
- In-dashboard case studies ("what's working")
- Weekly digest emails

### 20.4 Expansion revenue

- Materials usage scales with engagement
- Additional integrations as add-ons
- Multi-mailbox/multi-sender premium
- Team seats above included tier

### 20.5 Churn signals

- No leads added 14 days → at risk
- Quota hit but not renewed → at risk
- Sentinel rejection rate spiking → quality concern
- Trigger automated success outreach

---

## 21. Operational Runbooks (required by Phase 4 exit)

`.md` files in `/docs/runbooks/`, each tested at least once in staging:

1. **deploy.md** — staging-to-production promotion
2. **rollback.md** — code/database/Modal rollback
3. **incident-response.md** — P0/P1/P2 classification, response SLO, escalation
4. **database-recovery.md** — Neon PITR restore with tested scenarios
5. **secrets-rotation.md** — keys, OAuth, webhook secrets, DB passwords
6. **tenant-offboarding.md** — GDPR-compliant export + deletion
7. **cost-cap-exceeded.md** — quota response
8. **abuse-detection-response.md** — investigation + suspension
9. **anthropic-outage-failover.md** — OpenAI fallback + tenant comms
10. **modal-outage-failover.md** — in-process execution fallback

**On-call (Phase 4+):** primary rotates weekly. P0 → 15-min response. P1 → 1hr. P2 → next business day.

---

## 22. Team & Capital Requirements (HONEST RECKONING)

The architecture specifies a product a team of 4–5 senior engineers builds in 9–12 months. Choose one of three paths:

### Path A — Founder mode (1–2 engineers, scoped down)

- Phase 1: 6 agents, single tenant, REST only, 3–5 months
- Phase 2–3: add rest when team grows
- Phase 4: 6+ months
- First commercial tenant: month 12–18
- Capital: $100–300k self-funded or seed
- Win: useful product, modest revenue

### Path B — Funded mode (3–5 engineers, full scope)

- Phase 1: all 13 agents in 3 months
- Phase 2–3: months 4–6
- Phase 4: months 7–9
- First commercial: month 9–12
- Capital: $1.5–3M seed/Series A
- Win: competitive against 11x/Artisan, venture outcome

### Path C — Hybrid (1–2 engineers + on-demand contractors)

- Founder shape, specific phases (Verifier evals, video infra, MCP/A2A) accelerated by 4–8 week contractor bursts
- 12-month cost: $300–600k
- Timeline between A and B
- Win: real product month 9–12, optional later raise

**Decision before Phase 1 starts.** Doc supports all three; choice determines which Phase 1 plan in §13 to execute.

**Recommended:** Path C unless Path B capital genuinely available.

---

## 23. For Claude Code — How to Work

1. **Read §0 first** (Quick Reference)
2. **Find relevant agent (§6) and quality bar (§17)** before implementing
3. **Schema first** (Pydantic backend, Zod frontend)
4. **Test first** (happy path before implementation)
5. **Run `ruff` + `mypy` + tests locally before commit**
6. **ADR for any architectural decision not covered** → `/docs/adr/NNN-title.md`
7. **Update CLAUDE.md when behavior changes**

### Implementation checklist — new endpoint

- [ ] Pydantic input + output schemas
- [ ] Agent method on relevant class
- [ ] REST + MCP + A2A adapters
- [ ] Modal function if long-running
- [ ] Unit tests (happy + error)
- [ ] Eval cases if LLM involved (reference §17 criteria)
- [ ] OpenAPI regenerated
- [ ] SDK regenerated
- [ ] Docs page on Mintlify
- [ ] Audit log entry
- [ ] Tenant cost recorded via wrapper
- [ ] Cost cap check enforced

### Implementation checklist — new app page

- [ ] Route under `/apps/app/app/`
- [ ] Server Component default
- [ ] Data via tRPC + TanStack Query
- [ ] Forms via React Hook Form + Zod
- [ ] Loading + error states
- [ ] Mobile-responsive
- [ ] SSE realtime where state changes
- [ ] Playwright E2E happy path

---

## 24. Open Decisions

- [ ] Final product name (replace "arcadia")
- [ ] Path A / B / C from §22 — must decide before Phase 1
- [ ] Pricing model at Phase 4.5
- [ ] Brand identity
- [ ] Legal entity (separate SPV recommended)
- [ ] Commercialization wedge (vertical/persona/geography)
- [ ] Phase 2 friendly tenants
- [ ] Language expansion timing (Phase 5+, demand-driven)
- [ ] Founder time commitment (golden-dataset + eval discipline = 8–12 hrs/week sustained)

---

## 25. Glossary

- **Tenant** — customer org (UUID, Clerk-backed)
- **Lead** — target contact at target company
- **Crew** — 13 core + 2 sub-builder agents
- **Transport** — REST | MCP | A2A
- **Surface** — web app | Telegram | future channel
- **Master deck/video template** — tenant's brand-compliant templates
- **Brand voice** — structured voice profile
- **ICP** — Ideal Customer Profile
- **Capability** — discrete work unit exposed over transport
- **Materials** — on-demand artifacts (deck, video)
- **Inspectable run trace** — UI showing every step, prompt, model, cost
- **Outcomes** — recorded results feeding prompt evolution
- **Quality bar** — §17 measurable criteria
- **Golden dataset** — §17.4 hand-crafted ground truth
- **Founder / Funded / Hybrid mode** — §22 execution paths

---

*Version 4 (9/10 plan). Last updated 2026-05-15. Decisions locked here cannot be changed without ADR.*
