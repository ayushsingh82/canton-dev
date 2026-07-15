## Development Fund Proposal

**Author:** Ayush Singh (Individual contributor, ayushsinghmi711@gmail.com)
**Status:** Draft
**Created:** 2026-06-11
**Label:** wallet-apps

---

## Abstract

The wallet SDK can now send Utility Registry tokens (pre-approval work currently in progress). Three practical problems remain once that lands: a wallet or exchange that runs its own provider party on a Canton Coin pre-approval — the deliberate choice any provider makes to capture the app reward on incoming transfers — has no SDK support for that provider's full lifecycle (creation, monitoring, renewal, reward accounting, decommissioning), the existing unified transaction-history read path lacks per-registry symbol/metadata enrichment, and there is no standard way for one Canton wallet to request a payment from another. This proposal addresses all three as a single coherent scope — the lifecycle around a token transfer, not just the transfer itself.

---

## Specification

### 1. Objective

The work in progress on Utility Registry token pre-approvals closes the send gap. It does not close the gaps around sending. Default-provider (validator operator) Canton Coin pre-approvals already auto-renew via validator-app automation, and Utility Registry (CIP56) pre-approvals have no expiry at all, so neither of those needs renewal tooling. What's still missing is support for wallets and exchanges that choose to be their own provider party on a Canton Coin pre-approval — a deliberate choice made specifically to capture the app reward on incoming transfers that use that pre-approval — who today have no SDK surface for creating, monitoring, renewing, accounting for, or decommissioning that provider relationship. Separately, the SDK's `listHoldingTransactions` already returns a registry-agnostic transaction history in one call, but it doesn't carry human-readable per-registry instrument metadata (symbol, display name) alongside it, so a caller still has to fetch that separately per registry and join it by hand. And when a merchant or dApp wants to ask a user to send a specific token, there is no Canton equivalent of a payment URI — every app invents a format, so wallets cannot interoperate on incoming payment requests.

These three gaps are a natural follow-on to the pre-approval work and are best handled together, because they share the same SDK surface area and the same integration tests.

### 2. Implementation Mechanics

**Non-default-provider pre-approval lifecycle.** Add a full self-provider lifecycle layer for wallets and exchanges that run their own provider party on Canton Coin pre-approvals to capture the app reward, rather than defaulting to the validator operator. The SDK supports creating a pre-approval with the caller as provider, tracks expiry via a `getExpiringPreapprovals(withinDays)` helper, provides a `renewPreapproval` call that re-uses the existing factory resolution logic from the pre-approval work, tracks and reports app rewards earned per pre-approval, forecasts the renewal fee the provider will pay, supports batch creation/monitoring/renewal for exchanges managing many external parties, exposes monitoring/alerting hooks (metrics/webhooks) for renewal and failure state, and handles renewal failures with retry/backoff. No new protocol; no new contracts. This does not touch default-provider Canton Coin pre-approvals (already auto-renewed by validator-app automation) or Utility Registry pre-approvals (no expiry, nothing to renew).

**Multi-registry transfer history enrichment.** `listHoldingTransactions` already reads at the ledger's standardized Token Standard interface level, so it returns a registry-agnostic transaction history in one call, with `instrumentId` per line item and amounts as fixed-point Daml decimals — no per-registry decimal conversion needed. This proposal adds a convenience layer that joins that already-unified stream with per-registry instrument metadata (symbol, display name) fetched via the existing `registriesToAssets` helper, and wraps it with consistent cursor pagination on top of ledger offsets. A caller gets one enriched list regardless of how many registries a user holds tokens in, without hand-joining metadata calls themselves.

**Payment request format.** Define a Canton payment URI format and ship a parser and builder in the SDK. The URI encodes receiver party ID, registry ID, instrument ID, amount, and optional fields (memo, expiry, nonce). Example: `canton:PARTY_ID?registry=REGISTRY_ID&instrument=INSTRUMENT_ID&amount=10.00&memo=invoice-42`. The builder accepts a token type, amount, and party and returns a valid URI. The parser returns a structured object that feeds directly into the transfer path. The spec is published as a short Canton Improvement Proposal.

**Upstream contribution and reference integration.** All three pieces are contributed as pull requests to `canton-network/wallet`, matching the repository's code style and test conventions. A reference integration runs end to end against real registry endpoints: create a non-default-provider Canton Coin pre-approval, watch it approach expiry, renew it, send a Utility Registry token via a parsed payment request, and verify the transfer appears in the unified history. Kept green in CI.

### 3. Architectural Alignment

All three pieces extend the official wallet SDK rather than building a parallel library. None changes the protocol or existing SDK APIs. Pre-approval expiry sits above the existing pre-approval path. Unified history sits above the existing registry read path. Payment request format introduces a new URI scheme but does not touch the ledger. Backward compatible in all three cases.

### 4. Backward Compatibility

Additive only. No changes to existing SDK APIs, on-ledger contracts, or runtime behavior.

---

## Milestones and Deliverables

### Milestone 1 — Non-Default-Provider Pre-Approval Lifecycle
- **Duration:** Weeks 1–10
- **Deliverables:** Full self-provider lifecycle (creation, monitoring, renewal, decommissioning) for non-default-provider Canton Coin pre-approvals in the SDK token namespace; app-reward accounting; renewal fee tracking; bulk operations for exchanges; monitoring/alerting hooks; renewal failure handling with retry/backoff; unit tests; upstream pull request to `canton-network/wallet`.

| Task | Hours |
|---|---|
| Research non-default-provider setup and app-reward mechanics | 8h |
| Implement pre-approval creation flow (caller as provider) | 14h |
| Implement `getExpiringPreapprovals(withinDays)` monitoring helper | 10h |
| Implement `renewPreapproval` call with factory resolution reuse | 14h |
| Implement decommission/cancel flow | 6h |
| App-reward accounting and reporting tooling | 14h |
| Renewal fee tracking/forecast helper | 8h |
| Bulk operations (batch create/monitor/renew) for exchanges | 14h |
| Monitoring/alerting hooks (metrics/webhooks) | 8h |
| Renewal failure handling with retry/backoff | 6h |
| Unit tests and edge case coverage | 10h |
| Upstream PR preparation and review cycles | 6h |
| Documentation and inline examples | 2h |
| **Milestone 1 Total** | **120h** |

---

### Milestone 2 — Multi-Registry Transfer History Enrichment
- **Duration:** Weeks 11–16
- **Deliverables:** Convenience layer joining the existing registry-agnostic `listHoldingTransactions` stream with per-registry instrument metadata (symbol, display name) via `registriesToAssets`; consistent cursor pagination on top of ledger offsets; upstream pull request.

| Task | Hours |
|---|---|
| Document existing `listHoldingTransactions` / `registriesToAssets` behavior and design enrichment layer | 10h |
| Implement metadata-enrichment layer joining transaction stream with per-registry instrument metadata | 25h |
| Per-registry metadata joins (symbols, display names, instrument IDs) | 20h |
| Pagination ergonomics on top of existing ledger offsets | 10h |
| Integration tests against Canton Coin and Utility Registry | 15h |
| Upstream PR preparation and review cycles | 10h |
| Documentation and runnable examples | 5h |
| **Milestone 2 Total** | **95h** |

---

### Milestone 3 — Payment Request Format and Reference Integration
- **Duration:** Weeks 17–22
- **Deliverables:** Canton payment URI spec published as a CIP draft; parser and builder in the SDK; upstream pull request; reference integration running end to end in CI.

| Task | Hours |
|---|---|
| Draft Canton payment URI CIP spec | 15h |
| Implement URI builder | 10h |
| Implement URI parser with structured output | 15h |
| End-to-end reference integration (expiry → payment request → history) | 25h |
| CI setup and green-build maintenance | 10h |
| Outreach and onboarding of 2 independent wallet teams | 10h |
| Upstream PR preparation and review cycles | 10h |
| Documentation and integration guide | 5h |
| **Milestone 3 Total** | **100h** |

---

### Milestone 4a — External Auditor Fee (Pass-Through)
- **Duration:** Weeks 23–26
- **Deliverables:** Engagement of an independent third-party security firm to audit all SDK additions; published audit report.

This is a direct pass-through to the external auditor. Estimated cost: USD 25,000–30,000, billed in CC at spot rate at time of engagement. This line is not development time — it is the auditor's fee paid to a firm outside this proposal.

### Milestone 4b — Developer Remediation
- **Duration:** Weeks 23–26 (concurrent with audit)
- **Deliverables:** Remediation of all critical and high findings; re-test sign-off with auditor.

| Task | Hours |
|---|---|
| Audit scope document preparation and briefing | 5h |
| Coordination with auditor during review | 5h |
| Remediation of critical and high findings | 15h |
| Remediation of medium findings | 10h |
| Re-test and sign-off with auditor | 5h |
| **Milestone 4b Total** | **40h** |

---

## Complete Project Roadmap

| Week | Milestone | Focus |
|---|---|---|
| 1 | M1 | Research non-default-provider setup and app-reward mechanics |
| 2–3 | M1 | Implement pre-approval creation flow and `getExpiringPreapprovals` helper |
| 4–5 | M1 | Implement `renewPreapproval` call and decommission/cancel flow |
| 6–7 | M1 | App-reward accounting and renewal-fee tracking tooling |
| 8 | M1 | Bulk operations for exchanges (batch create/monitor/renew) |
| 9 | M1 | Monitoring/alerting hooks and renewal failure handling/retries |
| 10 | M1 | Unit tests, upstream PR and docs — **M1 delivery** |
| 11 | M2 | Design metadata-enrichment layer over existing `listHoldingTransactions` |
| 12–13 | M2 | Implement metadata-enrichment layer |
| 14 | M2 | Per-registry metadata joins |
| 15 | M2 | Pagination ergonomics and integration tests |
| 16 | M2 | Upstream PR and docs — **M2 delivery** |
| 17–18 | M3 | CIP spec draft and URI builder |
| 19 | M3 | URI parser implementation |
| 20–21 | M3 | End-to-end reference integration and CI setup |
| 22 | M3 | Wallet team outreach, upstream PR, docs — **M3 delivery** |
| 23 | M4 | Audit scope prep and auditor briefing |
| 24–25 | M4 | Audit execution and remediation |
| 26 | M4 | Re-test, final report publication — **M4 delivery** |

**Total project duration:** 26 weeks
**Total build hours (M1–M3):** 315h at 2,000 CC/h
**Total remediation hours (M4b):** 40h at 2,000 CC/h

---

## Acceptance Criteria

- Non-default-provider pre-approval lifecycle (creation, monitoring, renewal, reward accounting, decommissioning) merged into `canton-network/wallet`.
- Unified history interface and per-registry adapters merged into `canton-network/wallet`.
- Payment request format published as a CIP draft; parser and builder merged into `canton-network/wallet`.
- Reference integration runs end to end against real registry endpoints and is kept green in CI.
- At least 2 independent wallet teams use the payment request parser and builder.
- External audit completed with all critical and high findings remediated; audit report published.

Acceptance is based on real SDK usage, not on artifact delivery alone.

---

## Funding

**Rate:** 2,000 CC/h (consistent across all development milestones)

### Build Budget (M1–M3)

| Milestone | Deliverable | Hours | Rate | CC |
|---|---|---|---|---|
| M1 — Non-default-provider pre-approval lifecycle | SDK helpers + upstream PR | 120h | 2,000 CC/h | 240,000 CC |
| M2 — Multi-registry transfer history enrichment | Metadata enrichment + upstream PR | 95h | 2,000 CC/h | 190,000 CC |
| M3 — Payment request format | CIP + SDK + reference integration | 100h | 2,000 CC/h | 200,000 CC |
| **Build Total** | | **315h** | | **630,000 CC** |

### Audit Budget (M4 — additional, on top of build)

| Line | Description | CC |
|---|---|---|
| M4a — External auditor fee | Pass-through to independent security firm | USD 25,000–30,000 in CC at spot rate |
| M4b — Developer remediation | 40h × 2,000 CC/h | 80,000 CC |
| **Audit Total** | | **80,000 CC + auditor fee** |

**Total Funding Request: 710,000 CC + auditor pass-through**

Payment for each milestone is triggered upon committee acceptance of that milestone's deliverables.

### Note on Audit Cost
The Canton Foundation has covered external audit costs for wallet-critical proposals in the past and is welcome to do so here as well. If the Foundation prefers to fund M4a (the auditor fee) directly rather than as a pass-through, the build budget of 630,000 CC and developer remediation of 80,000 CC remain unchanged.

### Volatility Stipulation
Project duration is under 6 months. If the timeline extends beyond 6 months due to committee-requested scope changes, remaining milestones will be renegotiated to account for CC price movement.

---

## Adoption and Go-to-Market

- All SDK changes land upstream, so every team using the wallet SDK gets them by default.
- The payment request format is proposed as a CIP so any wallet can implement it independently and remain interoperable.
- Direct outreach to wallet teams currently building on the pre-approval work, since payment lifecycle completeness is the next immediate problem for them.
- The reference integration gives teams a known-good path to copy.

---

## Maintenance and Sustainability

The SDK contributions are maintained inside `canton-network/wallet` under its existing process. The payment request CIP is maintained as a Canton standard. Ayush Singh maintains the reference integration and tracks token-standard changes including the V2 token standard.

---

## Team

**Ayush Singh** — Full Stack Blockchain Engineer and DevRel
GitHub: https://github.com/ayushsingh82 | Portfolio: https://0xayush.vercel.app/

3+ years building across payments, stablecoins, account abstraction, DeFi, privacy infrastructure, and cross-chain systems. 20+ Web3 hackathon wins and multiple ecosystem grants across Ethereum, Solana, NEAR, Canton, EigenLayer, Stellar, and others.

- **Canton Capital** — Private fund operations platform on Canton Network with proposal creation, voting, execution, and real-time treasury analytics built in Daml.
https://github.com/ayushsingh82/Canton-Capital

- **CantonPay** — Confidential payroll infrastructure on Canton using Daml, enabling employer-employee payroll workflows with privacy and compliance.
https://github.com/ayushsingh82/CantonPay

- **IncoPay** — Private x402 payment infrastructure where users sign once and make multiple confidential payment requests.
https://x.com/IncoPayment

- **Nexchange** — Cross-chain staking and intent-based infrastructure on NEAR.
https://x.com/nexchange_near

- **AshWallet** — Anonymous wallet generation using chain signatures, deriving addresses on Solana, NEAR, and EVM from a single NEAR account with ZCash swap support via NEAR intents.
https://github.com/ayushsingh82/AshWallet

Also worked with GOAT Network, NEAR DevHub, Router Protocol, Oraichain, and Triplora.

---

## Motivation

The wallet team is building Utility Registry token pre-approvals now. The moment that lands, three gaps become the next real problem for wallet teams: any wallet or exchange choosing to run its own provider party on a Canton Coin pre-approval to capture app rewards has no SDK support for that lifecycle, transaction history lacks per-registry metadata enrichment, and there is no interoperability on payment requests. Each gap has been raised in community channels by builders hitting it. Addressing them as a package means the ecosystem gets payment lifecycle completeness once rather than each wallet team solving each piece separately.

---

## Rationale

The three pieces share the same SDK surface and the same integration tests, so bundling them is more efficient than three separate proposals and easier for the committee to evaluate as a coherent scope. Each piece is independently useful but they are most valuable together — a wallet that can send any token, track its pre-approvals, show a unified history, and accept incoming payment requests has a complete feature set. The payment request format in particular has the most ecosystem leverage because it enables interoperability across any wallet that implements the transfer path, including wallets the committee has no direct visibility into.
