## Development Fund Proposal

**Author:** Ayush Singh (Individual contributor, ayushsinghmi711@gmail.com)
**Status:** Draft
**Created:** 2026-06-11
**Label:** wallet-apps

---

## Abstract

The wallet SDK can now send Utility Registry tokens (pre-approval work currently in progress). Three practical problems remain once that lands: Canton Coin pre-approvals with a non-default provider party (and Utility Registry token pre-approvals generally) need renewal automation the SDK doesn't provide, the existing unified transaction-history read path lacks per-registry symbol/metadata enrichment, and there is no standard way for one Canton wallet to request a payment from another. This proposal addresses all three as a single coherent scope — the lifecycle around a token transfer, not just the transfer itself.

---

## Specification

### 1. Objective

The work in progress on Utility Registry token pre-approvals closes the send gap. It does not close the gaps around sending. Default-provider (validator operator) Canton Coin pre-approvals already auto-renew via validator-app automation, so most Canton Coin wallets never hit silent expiry — but non-default-provider Canton Coin setups, and Utility Registry token pre-approvals generally, still need renewal automation the SDK doesn't provide. Separately, the SDK's `listHoldingTransactions` already returns a registry-agnostic transaction history in one call, but it doesn't carry human-readable per-registry instrument metadata (symbol, display name) alongside it, so a caller still has to fetch that separately per registry and join it by hand. And when a merchant or dApp wants to ask a user to send a specific token, there is no Canton equivalent of a payment URI — every app invents a format, so wallets cannot interoperate on incoming payment requests.

These three gaps are a natural follow-on to the pre-approval work and are best handled together, because they share the same SDK surface area and the same integration tests.

### 2. Implementation Mechanics

**Pre-approval expiry management.** Add an expiry-aware layer scoped to the cases that actually lack automatic renewal: non-default-provider Canton Coin pre-approvals, and Utility Registry token pre-approvals. The SDK tracks the expiry timestamp for each, surfaces a `getExpiringPreapprovals(withinDays)` helper, and provides a `renewPreapproval` call that re-uses the existing factory resolution logic from the pre-approval work. No new protocol; no new contracts. This is a bookkeeping layer over what the pre-approval work already does, not a replacement for the validator-operator auto-renewal that already covers default-provider Canton Coin pre-approvals.

**Multi-registry transfer history enrichment.** `listHoldingTransactions` already reads at the ledger's standardized Token Standard interface level, so it returns a registry-agnostic transaction history in one call, with `instrumentId` per line item and amounts as fixed-point Daml decimals — no per-registry decimal conversion needed. This proposal adds a convenience layer that joins that already-unified stream with per-registry instrument metadata (symbol, display name) fetched via the existing `registriesToAssets` helper, and wraps it with consistent cursor pagination on top of ledger offsets. A caller gets one enriched list regardless of how many registries a user holds tokens in, without hand-joining metadata calls themselves.

**Payment request format.** Define a Canton payment URI format and ship a parser and builder in the SDK. The URI encodes receiver party ID, registry ID, instrument ID, amount, and optional fields (memo, expiry, nonce). Example: `canton:PARTY_ID?registry=REGISTRY_ID&instrument=INSTRUMENT_ID&amount=10.00&memo=invoice-42`. The builder accepts a token type, amount, and party and returns a valid URI. The parser returns a structured object that feeds directly into the transfer path. The spec is published as a short Canton Improvement Proposal.

**Upstream contribution and reference integration.** All three pieces are contributed as pull requests to `canton-network/wallet`, matching the repository's code style and test conventions. A reference integration runs end to end against real registry endpoints: set up a pre-approval, watch it approach expiry, renew it, send a Utility Registry token via a parsed payment request, and verify the transfer appears in the unified history. Kept green in CI.

### 3. Architectural Alignment

All three pieces extend the official wallet SDK rather than building a parallel library. None changes the protocol or existing SDK APIs. Pre-approval expiry sits above the existing pre-approval path. Unified history sits above the existing registry read path. Payment request format introduces a new URI scheme but does not touch the ledger. Backward compatible in all three cases.

### 4. Backward Compatibility

Additive only. No changes to existing SDK APIs, on-ledger contracts, or runtime behavior.

---

## Milestones and Deliverables

### Milestone 1 — Pre-approval Renewal Automation (Non-Default-Provider & Utility Registry)
- **Duration:** Weeks 1–7
- **Deliverables:** `getExpiringPreapprovals` helper and `renewPreapproval` call in the SDK token namespace, scoped to non-default-provider Canton Coin pre-approvals and Utility Registry token pre-approvals; unit tests; upstream pull request to `canton-network/wallet`.

| Task | Hours |
|---|---|
| Confirm Utility Registry token renewal behavior with registry/wallet team; research non-default-provider path | 10h |
| Implement `getExpiringPreapprovals(withinDays)` — scoped to non-default-provider Canton Coin + Utility Registry tokens | 20h |
| Implement `renewPreapproval` call with factory resolution reuse | 20h |
| Unit tests — non-default-provider and Utility Registry edge cases | 15h |
| Upstream PR preparation and review cycles | 10h |
| Documentation and inline examples | 5h |
| **Milestone 1 Total** | **80h** |

---

### Milestone 2 — Multi-Registry Transfer History Enrichment
- **Duration:** Weeks 8–13
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
- **Duration:** Weeks 14–19
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
- **Duration:** Weeks 20–23
- **Deliverables:** Engagement of an independent third-party security firm to audit all SDK additions; published audit report.

This is a direct pass-through to the external auditor. Estimated cost: USD 25,000–30,000, billed in CC at spot rate at time of engagement. This line is not development time — it is the auditor's fee paid to a firm outside this proposal.

### Milestone 4b — Developer Remediation
- **Duration:** Weeks 20–23 (concurrent with audit)
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
| 1 | M1 | Confirm Utility Registry renewal behavior; research non-default-provider path |
| 2–3 | M1 | Implement `getExpiringPreapprovals` helper |
| 4–5 | M1 | Implement `renewPreapproval` call |
| 6 | M1 | Unit tests and edge cases |
| 7 | M1 | Upstream PR and docs — **M1 delivery** |
| 8 | M2 | Design metadata-enrichment layer over existing `listHoldingTransactions` |
| 9–10 | M2 | Implement metadata-enrichment layer |
| 11 | M2 | Per-registry metadata joins |
| 12 | M2 | Pagination ergonomics and integration tests |
| 13 | M2 | Upstream PR and docs — **M2 delivery** |
| 14–15 | M3 | CIP spec draft and URI builder |
| 16 | M3 | URI parser implementation |
| 17–18 | M3 | End-to-end reference integration and CI setup |
| 19 | M3 | Wallet team outreach, upstream PR, docs — **M3 delivery** |
| 20 | M4 | Audit scope prep and auditor briefing |
| 21–22 | M4 | Audit execution and remediation |
| 23 | M4 | Re-test, final report publication — **M4 delivery** |

**Total project duration:** 23 weeks
**Total build hours (M1–M3):** 275h at 2,000 CC/h
**Total remediation hours (M4b):** 40h at 2,000 CC/h

---

## Acceptance Criteria

- Pre-approval expiry tracking and renewal merged into `canton-network/wallet`.
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
| M1 — Pre-approval expiry management | SDK helpers + upstream PR | 80h | 2,000 CC/h | 160,000 CC |
| M2 — Multi-registry transfer history | Unified history + adapters | 95h | 2,000 CC/h | 190,000 CC |
| M3 — Payment request format | CIP + SDK + reference integration | 100h | 2,000 CC/h | 200,000 CC |
| **Build Total** | | **275h** | | **550,000 CC** |

### Audit Budget (M4 — additional, on top of build)

| Line | Description | CC |
|---|---|---|
| M4a — External auditor fee | Pass-through to independent security firm | USD 25,000–30,000 in CC at spot rate |
| M4b — Developer remediation | 40h × 2,000 CC/h | 80,000 CC |
| **Audit Total** | | **80,000 CC + auditor fee** |

**Total Funding Request: 630,000 CC + auditor pass-through**

Payment for each milestone is triggered upon committee acceptance of that milestone's deliverables.

### Note on Audit Cost
The Canton Foundation has covered external audit costs for wallet-critical proposals in the past and is welcome to do so here as well. If the Foundation prefers to fund M4a (the auditor fee) directly rather than as a pass-through, the build budget of 550,000 CC and developer remediation of 80,000 CC remain unchanged.

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

The wallet team is building Utility Registry token pre-approvals now. The moment that lands, three gaps become the next real problem for any wallet team that uses it: silent pre-approval expiry, no unified history across registries, and no interoperability on payment requests. Each gap has been raised in community channels by builders hitting it. Addressing them as a package means the ecosystem gets payment lifecycle completeness once rather than each wallet team solving each piece separately.

---

## Rationale

The three pieces share the same SDK surface and the same integration tests, so bundling them is more efficient than three separate proposals and easier for the committee to evaluate as a coherent scope. Each piece is independently useful but they are most valuable together — a wallet that can send any token, track its pre-approvals, show a unified history, and accept incoming payment requests has a complete feature set. The payment request format in particular has the most ecosystem leverage because it enables interoperability across any wallet that implements the transfer path, including wallets the committee has no direct visibility into.
