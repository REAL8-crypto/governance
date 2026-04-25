# REAL8 Community Agents Network — Technical Architecture

**SCF Build Award submission · Integration Track**
**Project:** REAL8 Community Agents: Financial Inclusion on Stellar, Across Latin America
**Applicant:** Hispanopedia t/a REAL8 (Spanish non-profit, Málaga)
**Version:** 1.2 · 2026-04-23

---

## 1. Executive Summary

REAL8 is a Stellar-native asset launched in 2018 with a mission of financial inclusion in underserved communities. This proposal describes what REAL8 will build if awarded an SCF Build Award: a **Community Agents Network** — a platform of independent, locally-established operators who buy and sell REAL8 for commission, bringing the asset to people who are cut off from conventional banking rails. The initial deployment is a **pilot network across Mexico, El Salvador, Venezuela, and Argentina** — a deliberately multi-country first phase that lets us validate the model under different regulatory, banking, and currency conditions before expanding to the rest of Latin America and, subsequently, to other regions where the same structural needs exist. Venezuela is included as a specifically enabled market now that OFAC's partial lifting of sanctions on the Banco Central de Venezuela (General License nº 57, April 2026) has removed the previous blocking legal risk for this kind of work.

The network is built on top of the existing REAL8 stack: wallet app, multi-chain bridge, SEP-2 federation server, MoneyGram integration, and Stripe checkout — all of which are already in production. With this Build Award, REAL8 will build the specific new surfaces needed to launch the agent network: onboarding, contract anchoring, on-chain activity monitoring, bonification payout, agent and admin panels, and fraud mitigations.

---

## 2. Project Scope

### In-scope deliverables (3-5 month window)

1. **Agent onboarding flow** — KYC intake, risk screening (OFAC SDN, PEP), approval workflow, Stellar account linking.
2. **Contract signing + on-chain anchoring** — PDF agreement signed off-chain, SHA-256 hash anchored on Stellar via memo transaction, plus SEP-10-style challenge that the agent signs with their account key to record acceptance on-chain.
3. **Agent panel** — self-service dashboard for each agent: operations, unique-client count, monthly bonification estimate, payout history, limits usage.
4. **Admin panel (REAL8 team)** — agent approval, suspension, audit, manual bonification review for large amounts, fraud-signal dashboard.
5. **On-chain activity indexer** — PostgreSQL-backed indexer that tracks agent Stellar accounts' outgoing REAL8 payments, classifies them (sale to client, internal transfer, excluded address), and maintains monthly aggregates.
6. **Monthly bonification job** — scheduled job computing 5% bonification per agent, verifying the 25-unique-client threshold, and automatically paying in REAL8 to the agent's Stellar account (with manual audit gate above a configurable threshold).
7. **Fraud mitigation layer** — account-age minimum, minimum transaction size, suspicious-pattern detection, exclusion allowlist for non-client transfers.
8. **Bilingual UX (Spanish/English)** — following the existing localization conventions of the REAL8 wallet.

### Explicitly out-of-scope

- Cash custody or escrow — the agent operates with their own REAL8 inventory, REAL8.org never holds their fiat or tokens.
- Client-side KYC — the agent is responsible for knowing their client.
- Soroban governance contracts — considered for a v2, not part of this Build Award.

---

## 3. High-Level Architecture

```
┌───────────────────────┐          ┌───────────────────────────┐
│   agent.real8.org     │          │    admin panel (w.real8)  │
│   (agent dashboard)   │          │  (REAL8 team operations)  │
└──────────┬────────────┘          └────────────┬──────────────┘
           │                                    │
           ▼                                    ▼
┌───────────────────────────────────────────────────────────────┐
│             api.real8.org  (Express + TypeScript)             │
│  /agents · /agents/:id · /agents/:id/bonification · /admin/*  │
└──────────────────────────────┬────────────────────────────────┘
                               │
      ┌────────────────────────┼────────────────────────┐
      ▼                        ▼                        ▼
┌──────────┐          ┌─────────────────┐      ┌─────────────────┐
│PostgreSQL│          │ OFAC / SDN API  │      │ Stellar Horizon │
│ real8_   │          │ (Treasury gov)  │      │  (public node)  │
│ bridge   │          └─────────────────┘      └─────────────────┘
└──────────┘
      │
      ▼
┌────────────────────────────────────────────────────────────────┐
│                     Background services                         │
│                                                                 │
│   agent-activity-indexer    monthly-bonification-job            │
│   (follows agent accounts,  (runs 1st of month,                 │
│    classifies payments)      pays 5% to qualifying agents)      │
└────────────────────────────────────────────────────────────────┘
      │
      ▼
┌────────────────────────────────────────────────────────────────┐
│         Stellar Public Network                                  │
│                                                                 │
│   REAL8 issuer · Agent accounts · Bonification distribution     │
│   Federation (agent@real8.org) · Contract-anchor transactions   │
└────────────────────────────────────────────────────────────────┘
```

All new components sit behind the existing `api.real8.org` Express backend and share its PostgreSQL database (`real8_bridge`), authentication layer, MaxMind geoblock middleware, rate limiters, and Postmark email service. The indexer and bonification job run as additional PM2 processes alongside the existing `real8-moneygram-api` and `bridge-automation` services.

---

## 4. Stellar Building Blocks Integrated

The Integration Track is specifically for projects "incorporating existing Stellar building blocks." This proposal integrates:

| Building block | Use |
|---|---|
| **SEP-2 Federation** | Every agent gets a federation address `agent@real8.org` via the existing federation server (already in production, `federation_records` table). Clients transact using memorable addresses instead of raw G-accounts. |
| **SEP-10 Web Auth** | Agent login to the panel uses SEP-10 authentication, reusing the existing `/auth` endpoint and signing flow already used for MoneyGram. Adapted to add an "acceptance of terms" challenge XDR that records contract signing on-chain. |
| **Stellar Native Multisig** | Agent Stellar accounts can optionally be configured as 2-of-3 multisig for enhanced security of larger inventories — native multisig, no smart contract overhead. |
| **Horizon Streaming API** | The agent-activity indexer uses Horizon's `/accounts/:id/payments` endpoint with cursor persistence (pattern reused from `stellarMonitorService.ts` already in production for the bridge). |
| **Stellar Transaction Memos** | Contract anchoring writes the SHA-256 of the signed PDF agreement into the `memo_hash` field of a transaction between REAL8 and the agent account, producing a verifiable, timestamped record with no extra infrastructure. |
| **Classic REAL8 payments** | The bonification payout is a standard REAL8 asset payment from a dedicated distribution account to the agent account — reusing the sequence-isolated distribution pattern already in use for Stripe checkout (account `GCK5CADT4FYE4I6ILZQ42KB6AGHYZ7UVRVX3ERGNDN7ZHYO5BBPQ7UZQ`). |

These are **all production-grade, Stellar-native primitives**. The project ships no custom Stellar protocol extensions; it composes existing SEPs and operations into a cohesive financial-inclusion network.

---

## 5. Data Model

New PostgreSQL tables in the existing `real8_bridge` database:

```sql
-- Agent profile
CREATE TABLE agents (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  stellar_address TEXT UNIQUE NOT NULL,          -- G...
  federation_name TEXT UNIQUE,                   -- registered in federation_records
  country_iso2 CHAR(2) NOT NULL,
  kyc_status TEXT NOT NULL,                      -- pending | approved | suspended | revoked
  kyc_documents JSONB,                           -- references to S3/Storage keys
  contract_anchor_tx TEXT,                       -- Stellar tx hash of SHA-256 memo
  contract_pdf_sha256 TEXT,
  bonification_auto_max_usd NUMERIC(10,2),       -- above this, manual audit required
  daily_volume_cap_usd NUMERIC(10,2),
  monthly_volume_cap_usd NUMERIC(10,2),
  created_at TIMESTAMPTZ DEFAULT NOW(),
  approved_at TIMESTAMPTZ,
  suspended_at TIMESTAMPTZ
);

-- Indexer state — one row per agent
CREATE TABLE agent_activity_cursor (
  agent_id UUID PRIMARY KEY REFERENCES agents(id),
  last_horizon_cursor TEXT,
  last_indexed_at TIMESTAMPTZ
);

-- Classified outgoing payments from agent account
CREATE TABLE agent_transactions (
  id BIGSERIAL PRIMARY KEY,
  agent_id UUID REFERENCES agents(id),
  stellar_tx_hash TEXT UNIQUE NOT NULL,
  destination_account TEXT NOT NULL,
  real8_amount NUMERIC(20,7) NOT NULL,
  usd_equivalent NUMERIC(20,2),                  -- snapshot at indexing time
  classification TEXT NOT NULL,                  -- client_sale | internal_transfer | exchange | excluded
  classification_reason TEXT,
  counted_for_bonification BOOLEAN NOT NULL,
  ledger_closed_at TIMESTAMPTZ NOT NULL,
  indexed_at TIMESTAMPTZ DEFAULT NOW()
);

-- Monthly aggregates and bonification
CREATE TABLE agent_monthly_summary (
  id BIGSERIAL PRIMARY KEY,
  agent_id UUID REFERENCES agents(id),
  month DATE NOT NULL,                           -- first day of month
  eligible_sales_real8 NUMERIC(20,7) DEFAULT 0,
  eligible_sales_usd NUMERIC(20,2) DEFAULT 0,
  unique_clients INT DEFAULT 0,
  unique_client_threshold INT NOT NULL,          -- 5/15/20/25 gradual
  threshold_met BOOLEAN DEFAULT FALSE,
  bonification_real8 NUMERIC(20,7),
  bonification_status TEXT,                      -- pending | approved | paid | held_for_audit
  bonification_paid_tx TEXT,
  audit_notes TEXT,
  UNIQUE(agent_id, month)
);

-- Exclusion allowlist (exchanges, other agent wallets, REAL8 treasury, etc.)
CREATE TABLE agent_exclusion_addresses (
  stellar_address TEXT PRIMARY KEY,
  kind TEXT NOT NULL,                            -- exchange | agent_own | real8_treasury | other
  notes TEXT,
  added_at TIMESTAMPTZ DEFAULT NOW()
);
```

Row-level locking (`SELECT ... FOR UPDATE`) is used on `agent_monthly_summary` during the bonification job, following the same atomic pattern adopted for the bridge release limits (`bridge_releases` table, api v1.6.1).

---

## 6. Core Flows

### 6.1 Agent onboarding

1. Prospective agent registers via `agent.real8.org/signup`: email, country, Stellar address (new or existing).
2. KYC documents uploaded to encrypted object storage (AES-256-GCM envelope, matching the encryption pattern already used for airdrop private keys, api v1.6.4).
3. OFAC SDN list screening via Treasury's public API.
4. REAL8 team reviews in admin panel, approves or rejects. On approval:
   - Federation record `agent_<slug>@real8.org` created in `federation_records`.
   - PDF contract generated with agent's legal + Stellar data pre-filled.
5. Agent signs PDF off-chain (DocuSign or equivalent — decided in kickoff).
6. SHA-256 of signed PDF computed, written to a Stellar tx memo from the REAL8 treasury to the agent's account, along with a SEP-10-style challenge XDR the agent signs to record their acceptance.
7. Contract anchor tx hash stored in `agents.contract_anchor_tx`. Agent is now live.

### 6.2 Client sale (observed, not mediated)

REAL8.org does **not** mediate the transaction. The agent sells REAL8 from their own Stellar account to a client who has paid them in local fiat. From our side:

1. `agent-activity-indexer` polls Horizon `/accounts/:id/payments` for each approved agent (30s cadence, cursor-persisted).
2. For each outgoing REAL8 payment:
   - Check destination against `agent_exclusion_addresses` → classify as `internal_transfer` or `exchange` if matched (doesn't count).
   - Check destination's age (trustline creation + first-op timestamp via Horizon): if either < 30 days, classify as `pending_review` (doesn't count this cycle, flagged for audit).
   - Check destination's prior tx with non-agent counterparty: if none, flagged for fraud review.
   - Check amount: if < configurable minimum (default 50 REAL8 ≈ $5), classify as `below_threshold` (doesn't count).
   - Otherwise classify as `client_sale`, insert into `agent_transactions` with `counted_for_bonification = true`.
3. Unique-client aggregation: a `HyperLogLog` sketch per agent per month (using PostgreSQL's `hll` extension) for accurate unique-account counting without storing raw lists longer than needed.

### 6.3 Monthly bonification cycle

On the 1st of each month at 03:00 UTC:

1. For each approved agent, aggregate the prior month's `agent_transactions` where `counted_for_bonification = true`.
2. Apply the progressive unique-client threshold: 5 (month 1) → 15 (2) → 20 (3) → 25 (month 4+).
3. If threshold met: compute 5% bonification in REAL8, using the end-of-period REAL8/USD rate snapshot from `pricingService`.
4. If bonification > `agent.bonification_auto_max_usd` (default $500): set status `held_for_audit`, notify admin.
5. Otherwise: create an atomic payout operation — row-lock `agent_monthly_summary`, submit Stellar payment from the bonification distribution account, store tx hash, mark `paid`.
6. Send per-agent email via Postmark (bilingual ES/EN) with the detailed breakdown.

### 6.4 Suspension and appeal

Admin can suspend any agent from the admin panel. Suspension halts indexing and pauses any pending bonification. Appeals recorded as audit-log rows for compliance trail.

---

## 7. Anti-Fraud Design

The most significant security risk is that an agent could create fake Stellar destination accounts and self-pay to earn the 5% bonification on non-existent activity. The attack cost without mitigation is ~3 EUR for 25 fake accounts, and the payout could be arbitrarily large. Mitigations layered in this architecture:

1. **Account age minimum (30 days)** on destination accounts — raises the attack cost and introduces a 30-day window for detection.
2. **Prior-counterparty requirement** — destination must have had at least one prior operation with a non-agent counterparty, breaking pure self-funnel schemes.
3. **Minimum transaction size** — below 50 REAL8 (≈ $5) doesn't count, preventing spam-microtransaction inflation.
4. **Configurable auto-payout ceiling** — default $500; above that, manual admin review.
5. **Pattern detection** — clusters of destination accounts with shared creation date, shared IP (for any REAL8-owned surface they touched), or sequential Stellar ID runs are flagged.
6. **Agent deposit (optional)** — 100 REAL8 refundable good-faith deposit at onboarding, forfeited if fraud detected; can be toggled per agent by admin.

Mitigations are implementation-level and configurable — the specific combination to activate will be confirmed with the project's administrative leads during the kickoff milestone.

---

## 8. Technical Stack

| Layer | Technology | Notes |
|---|---|---|
| Frontend (agent + admin panels) | React 19 + Vite 8 + MUI 7 + TypeScript 5.8 | Matches the existing `app.real8.org` wallet stack; shared component library |
| Backend | Express 5.2 + TypeScript 5.8 on Node 20 | Same codebase as `api.real8.org` — new routes added, no separate service |
| Database | PostgreSQL 15 | Same `real8_bridge` instance — new tables, existing infrastructure |
| Stellar SDK | `@stellar/stellar-sdk 14.6.1` | Already in use for bridge, federation, pricing |
| Background jobs | PM2 processes + node-cron | Pattern proven with `bridge-automation` |
| Email | Postmark via existing `emailService` singleton | Bilingual ES/EN |
| Geoblock | Existing MaxMind-based middleware (api v1.6.0) | Country and hosting-ASN signals already in production |
| Hosting | Webdock VPS (`api.real8.org`) | Current production infrastructure |

No new infrastructure required. All work composes cleanly into the existing stack.

---

## 9. Deliverables & Milestones

| Milestone | Duration | Deliverable |
|---|---|---|
| M1 — Kickoff & schema freeze | Weeks 1-2 | Final fraud-mitigation config with the administrative leads; DB schema migration; admin user roles |
| M2 — Agent onboarding + contract anchor | Weeks 3-5 | Signup flow, KYC intake, OFAC screening, PDF generation, on-chain anchor + SEP-10 acceptance |
| M3 — Activity indexer + exclusion list | Weeks 6-8 | Horizon poller with cursor persistence, classification engine, exclusion admin UI |
| M4 — Agent panel + federation | Weeks 9-11 | Agent dashboard (operations, unique-client progress, bonification estimate), `agent@real8.org` issuance |
| M5 — Admin panel + bonification job | Weeks 12-14 | Admin review queue, manual audit workflow, monthly bonification cron + automated payout |
| M6 — Hardening, external audit, pilot launch | Weeks 15-18 | External security review, penetration test, fraud-scenario tabletop, bilingual UX QA, onboard first pilot agents across Mexico, El Salvador, Venezuela and Argentina |

Final deliverable: a live, auditable agent network with pilot agents onboarded and transacting REAL8 across all four pilot markets (Mexico, El Salvador, Venezuela, Argentina), positioning REAL8 to expand the network to additional Latin American countries and beyond in the quarters that follow.

---

## 10. Team, Sponsor, and Technical Partner

REAL8 is the trading name of **Hispanopedia**, a Spanish non-profit organisation registered in Málaga. The REAL8 project has been shipping continuously on Stellar since 2018. Governance, funding compliance, and public accountability flow through the non-profit entity.

### Core team (12 members across three continents)

| Name | Role | Location |
|---|---|---|
| Rafael Minuesa (Rafael Martinez Minuesa) | CEO & Web Development · MoneyGram ToS signatory | Marbella, Spain |
| Anselmo Rodríguez Manzo | Treasurer · multi-country operations · Community Agents Network design lead | Baton Rouge, United States |
| Christophe de Landtsheer | Chief Financial Officer | Brussels, Belgium |
| Lauren Pollack | VP of Marketing | New York, United States |
| Eduardo Marín Aznárez | Systems Administrator | Edinburgh, Scotland |
| Emmanuel Chávez | Multimedia & Digital Developer | Michoacán, Mexico |
| Ángel Benzal | Investor Relations | Cartagena, Spain |
| Cecilia Velazquez Vertti | Communications Director | Puebla, Mexico |
| Madeleine Arévalo Camacho | VP of Public Relations | Santa Marta, Colombia |
| Lucinda Olaseinde | Social Media Manager | Florence, Italy |
| Carlos Fernández Andrade | Strategic Historical Advisor | Seville, Spain |
| Daiki Fujii | Blockchain & Web3 Developer | Tokyo, Japan |

The team is distributed across Europe, the Americas, and Asia, with on-the-ground presence in two of the four pilot markets (Mexico, via Emmanuel Chávez and Cecilia Velazquez Vertti), and adjacent Latin-American coverage through Madeleine Arévalo Camacho (Colombia). Further country-lead coverage for the remaining pilot markets (El Salvador, Venezuela, Argentina) is staffed through the extended partner network described below.

Full team profiles: [real8.org/en/our-team/](https://real8.org/en/our-team/)

### Sponsor — Hispanopedia

Hispanopedia is the Spanish non-profit that sponsors and legally hosts the REAL8 project. As a non-profit, REAL8's financial-inclusion and educational mission is embedded in its charter, not layered onto a for-profit entity. Agreements, grants, and compliance obligations are signed by Hispanopedia.

### Technical partner — ProWoos

**ProWoos** is REAL8's founding technical partner, contributing a team of **30+ experienced professionals** across nine key locations worldwide, including Buenos Aires, Davao, Kraków, Madrid, Marbella, New York, Porto Alegre, and Santiago de Chile. ProWoos anchors REAL8's engineering capacity and contributes the continuous-delivery discipline that has kept the project shipping for eight years. The Buenos Aires and Latin-American ProWoos presence is directly relevant to this proposal: it places technical staff on the ground in pilot-market time zones throughout the build and rollout phases.

### Extended network

REAL8 also collaborates with cultural associations, NPO partners, academics, financial-institution contacts, and government-agency representatives whose programmes align with the mission. These relationships support in-country activation, local legal review, and community engagement across the pilot markets.

### External relationships relevant to this proposal

- **MoneyGram Payment Systems, Inc.** — commercial counterparty for fiat cash-in/cash-out via USDC in the REAL8 Wallet. Contacts: Sergio Garzón Forero (commercial) and Tim Dugan (technical). Terms of Service v3 signing scheduled for late April 2026.
- **Stellar Development Foundation** — the ecosystem we build on; SCF is the award program this proposal addresses.

---

## 11. Risks & Dependencies

- **Legal review — per pilot country:** each of the four pilot markets (Mexico, El Salvador, Venezuela, Argentina) has its own regulatory context and is covered in parallel by the respective country leads. Venezuela's case is specifically unblocked by OFAC General License nº 57 (April 2026), which lifted the prior sanctions on the Banco Central de Venezuela; country-specific legal confirmation remains a gating item in every market before live pilot.
- **KYC provider selection:** deferred to M1. Candidate vendors: Sumsub, Onfido. Fallback: in-house document collection + manual review for the pilot window.
- **DocuSign or equivalent:** small external dependency for off-chain PDF signing. Can be replaced with any signing provider that returns a canonical PDF hashable to SHA-256.
- **Dependency on upstream Stellar infrastructure:** all operations use public Horizon; SDF-operated Horizon is the primary target with community-run backups for redundancy.

---

## 12. Relationship to the Existing REAL8 Stack

What REAL8 would deliver under this Build Award extends — not replaces — REAL8's current production infrastructure. Components already shipped that this proposal builds on:

- **app.real8.org** (wallet, v4.4.0) — shared component library and authentication primitives.
- **api.real8.org** (backend, v1.7.0) — the new agent routes live here; existing SEP-2 federation, SEP-10 auth, MaxMind geoblock, Postmark mail, Stripe checkout, pricing feeds are all reused.
- **real8-bridge** (multi-chain wREAL8) — the agent panel exposes wREAL8 bridging for agents operating in multi-chain regions, reusing the existing bridge automation.

The net effect is a cohesive, Stellar-anchored financial-inclusion platform where the agent network becomes the human-facing last mile of an existing, mature, Stellar-native technology stack.
