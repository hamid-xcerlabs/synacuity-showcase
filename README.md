# Synacuity — AI-Native Restaurant Intelligence & Action Platform

> **Source-free showcase.** Production source code is maintained privately under a client IP agreement. This repository documents the architecture, engineering decisions, and system design behind the platform.

---

## What Is Synacuity?

Synacuity is an **AI-native intelligence and action layer** built for multi-location restaurant chains.

It sits above the existing restaurant technology ecosystem — connecting fragmented guest signals, location data, and operational signals into a unified intelligence system that understands what is happening, explains why, prioritizes what matters, and executes approved actions.

**Current production deployment:** Enterprise QSR chain — 2,400+ locations across two markets.

---

## The Problem

Large restaurant chains receive tens of thousands of guest reviews across hundreds of locations — every month. The existing approach:

- Manual review monitoring across disconnected platforms
- No structured intelligence — just raw text
- No location-level pattern detection
- Responses drafted manually or left unanswered
- No way to connect guest feedback to operational cause
- No executive visibility across the network

The result: **signal buried in noise, action delayed or absent, revenue at risk.**

---

## The Solution Architecture

```
EXTERNAL SOURCES (Google Business Profile, Yelp, TripAdvisor, DoorDash...)
          │
    CONNECTOR LAYER
          │
    INGESTION LAYER (normalize / validate / dedupe / idempotency)
          │
    UNIFIED DATA LAYER
    ┌─────────────────────────────────────────┐
    │  Organization → Brand → Location        │
    │  Source → Review → Guest → Action       │
    └─────────────────────────────────────────┘
          │
    INTELLIGENCE ENGINE
    ┌─────────────────────────────────────────┐
    │  sentiment · topics · issue type        │
    │  severity · priority · signals          │
    │  escalation · recovery · trends         │
    └─────────────────────────────────────────┘
          │
    ┌─────┴──────┬──────────────┐
    ▼            ▼              ▼
GUEST VOICE  REPUTATION    LOCATION
INTELLIGENCE INTELLIGENCE  INTELLIGENCE
    │            │              │
    └─────┬──────┴──────────────┘
          ▼
    INSIGHTS ENGINE
          │
    ACTION CENTER
    ┌─────┬───────┬──────────────┐
    ▼     ▼       ▼              ▼
APPROVE  EDIT  REGENERATE  WRITE OWN
          │
      AI AUDIT
          │
       PUBLISH → Google Business Profile
          │
    OUTCOME / AUDIT TRAIL
```

**Core loop:** Signals → Context → Intelligence → Prioritize → Decide → Act → Outcome → Learn

---

## Production Numbers (Live System)

| Metric | Value |
|--------|-------|
| Reviews ingested | **7,060+** current reviews |
| Locations monitored | **2,427** across 2 markets |
| AI intelligence records | **6,100+** |
| AI calls processed | **558** tracked calls |
| Responses published to Google | **58** live replies |
| Organizations (tenants) | **2** |
| AI cost per 1,000 reviews analyzed | **~$6.40** |
| Pipeline latency | Sub-2-minute (Pub/Sub → processed) |

---

## Technical Stack

### Frontend
- **Next.js 15** (App Router) + TypeScript — end-to-end type safety
- **Tailwind CSS v4** + **shadcn/ui** (Radix UI primitives)
- **TanStack Table v8** — server-side pagination, filtering, sorting
- **Recharts** — sentiment trends, rating distributions, location health charts
- **Zustand** + **TanStack Query** — client state + server state separation
- **Supabase Realtime** — live updates on inbox and action states

### Backend
- **Next.js Route Handlers** — API layer
- **Trigger.dev v4** — background job orchestration (review pull, AI processing, publish pipeline)
- **Supabase PostgreSQL** — primary data store with RLS on all tables
- **Supabase Edge Functions** — serverless event handling
- **Zod** — runtime schema validation end-to-end

### AI Layer
- **Claude Sonnet (claude-sonnet-4-6)** — review intelligence + response generation + audit
- **Prompt caching (1h ephemeral)** — on static instruction blocks to reduce cost
- **AI usage tracking** — per-call cost, tokens, feature, model version logged to DB
- **Per-tenant AI toggle** — Review Intelligence can be enabled/disabled per organization

### Infrastructure
- **Google Business Profile API v4** — review retrieval + reply publishing
- **Google Cloud Pub/Sub** — real-time review change notifications
- **Vercel** — frontend + API deployment
- **Trigger.dev Cloud** — background job runtime

---

## Database Design

**17 tables**, multi-tenant from day one. Core entities:

```
organizations
  └── brands
        └── locations
              └── source_locations (per-source metadata)

sources
  └── reviews (canonical, is_current versioning)
        └── review_intelligence (32 AI-derived fields)
        └── review_actions (state machine)
        └── responses (versioned response history)
              └── response_audits

users → profiles
ai_usage_events (per-call cost tracking)
ai_model_pricing (versioned pricing, frozen at call time)
platform_settings
admin_audit_log
account_org_mapping
```

**Key design decisions:**

- **Source-agnostic canonical review** — `review_id` is our internal identity; `gbp_review_id` is integration metadata. New sources (Yelp, TripAdvisor, DoorDash) plug in without schema changes.
- **`is_current` versioning** — review updates create new versions, preserving audit history while dashboards show only current state.
- **RLS on all tables** — row-level security enforced at DB layer, not just application layer.
- **No client-hardcoded tables** — `organizations`, `brands`, `locations` — not `kfc_reviews`, `kfc_locations`.
- **AI cost frozen at call time** — `ai_model_pricing` table with versioned records; cost calculated at the moment of the API call, not recalculated later.

---

## AI Intelligence Schema (32 fields per review)

Each review is analyzed by Claude and produces structured intelligence:

```
sentiment (positive / neutral / negative)
sentiment_confidence
primary_topic
secondary_topics[]
issue_type
issue_severity (none / low / medium / high / critical)
service_channel
customer_type_signal
resolution_mentioned
resolution_description
recovery_signal          ← feeds future Customer Recovery domain
escalation_signal        ← triggers priority elevation
competitor_mentioned     ← feeds future Competitive Intelligence domain
competitor_names[]
return_intent
customer_requested_action
ai_priority (low / medium / high / urgent)
ai_action_recommendation
key_issue_summary
key_positive_summary
positive_signals[]
negative_signals[]
mentioned_entities[]
service_channels_mentioned[]
...
```

These fields are Guest Voice signals today. They are designed to feed future intelligence domains (Operations, Competitive, Recovery, Campaign) without schema migration.

---

## Pipeline: Review Lifecycle

```
Google Business Profile
        │
   Pub/Sub notification
        │
   Trigger.dev webhook receiver
        │
   Fetch authoritative review from GBP API
        │
   Normalize → validate → dedupe
        │
   Persist canonical review (is_current versioning)
        │
   Trigger AI processing (Claude Sonnet)
        │
   Store review_intelligence (32 fields)
        │
   Create review_action (state: pending)
        │
   Surface in Guest Inbox
        │
   Manager: Approve / Edit / Regenerate / Write Own
        │
   AI Audit (score, brand fit, risk, improvement)
        │
   Publish to Google Business Profile API
        │
   Record outcome + audit trail
```

---

## Action State Machine

```
new → needs_review → ai_draft_ready → awaiting_approval
                                              │
                          ┌───────────────────┼───────────────────┐
                          ▼                   ▼                   ▼
                       approved            edited            regenerate
                          │                   │                   │
                          └───────────────────┼───────────────────┘
                                              ▼
                                          ai_audit
                                              │
                                          publishing
                                              │
                                   ┌──────────┴──────────┐
                                   ▼                     ▼
                                published             failed
```

Human approval required before any public reply is published. AI does not self-publish.

---

## AI Audit System

Every response — AI-generated, manager-edited, or manager-written — passes through an audit before publication:

| Dimension | Description |
|-----------|-------------|
| Overall Score | 0–100 governance score |
| Brand Fit | Matches brand tone and guidelines |
| Issue Addressed | Directly responds to the guest's complaint |
| Tone Appropriate | Professional and context-appropriate |
| Empathy | Acknowledges guest experience |
| Accuracy | No factual errors or false promises |
| Risk Level | low / medium / high / critical |
| Suggested Improvement | Specific actionable improvement if score < threshold |

This gives: **human control + AI governance** — neither fully automated nor fully manual.

---

## Intelligence Domains (MVP → Roadmap)

| Domain | MVP Status |
|--------|-----------|
| Guest Voice Intelligence | ✅ Built |
| Reputation Intelligence | ✅ Built |
| Location Intelligence (review-derived) | ✅ Built |
| Executive / Network Overview | ✅ Built |
| AI Response + Action Center | ✅ Built |
| Unified Customer Intelligence | 🔲 Data model ready |
| Operations Intelligence | 🔲 Roadmap |
| Campaign Intelligence | 🔲 Roadmap |
| Competitive Intelligence | 🔲 Roadmap |
| Local Presence / AI Search Intelligence | 🔲 Roadmap |
| Revenue Intelligence | 🔲 Roadmap |
| Predictive Intelligence | 🔲 Roadmap |
| Customer Recovery | 🔲 Roadmap |

Architecture is designed so future domains consume the same data primitives — no platform rebuild required.

---

## Key Engineering Decisions

**1. Canonical data layer over source-specific tables**
Reviews from Google, Yelp, DoorDash all normalize to the same internal `reviews` schema. The intelligence engine doesn't care about the source.

**2. Multi-tenant from day one, single-tenant UX for MVP**
Database has `organization_id` on every row, RLS enforced at DB layer. Adding a second restaurant client requires zero schema changes.

**3. Trigger.dev over n8n**
n8n was the original prototype. Replaced with Trigger.dev v4 for proper TypeScript-native background jobs, retry logic, observability, and production reliability.

**4. Pub/Sub over polling**
Google Pub/Sub sends real-time notifications on review create/update. The system fetches the authoritative review from GBP API on notification rather than treating the notification payload as the source of truth.

**5. Prompt caching on intelligence calls**
Static system instruction block marked `cache_control: ephemeral (1h)`. Reduces per-call cost on the high-volume review intelligence feature.

**6. AI cost tracking at call time**
`ai_model_pricing` table stores versioned pricing. Cost is calculated and frozen at the moment of each API call — not subject to future pricing changes.

**7. Response versioning**
Every response draft (AI or manager) is stored with a version. Full history preserved for audit, compliance, and future training data.

---

## What This Is Not

- Not a review management tool that generates bulk replies
- Not a chatbot with an "Ask AI" box
- Not a replacement for POS, CRM, or loyalty systems
- Not hardcoded to any single restaurant brand

It is an intelligence and action layer that sits above the existing restaurant technology stack.

---

## Business Context

**Company:** Synacuity (Synergy + Acuity) — Synacuity LLC, Texas  
**Stage:** Production MVP, enterprise client deployed  
**Target market:** Multi-location QSR / fast-casual restaurant groups  
**Pricing model:** Platform subscription (per-location)  
**Live at:** [app.synacuity.com](https://app.synacuity.com)

---

## Source Code

Production source code is maintained in a private repository under a client IP agreement.

**For technical interviews or recruiter review:** Code walkthrough available on request. Contact: [hamid@xcerlabs.com](mailto:hamid@xcerlabs.com)

---

*Built by [Hamid Reyes](https://hamid.xcerlabs.com) — AI Systems, Automation & Software*
