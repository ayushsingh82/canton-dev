## Development Fund Proposal

**Author:** Ayush Singh (Individual contributor, ayushsinghmi711@gmail.com)
**Status:** Draft
**Created:** 2026-06-11
**Label:** wallet-apps

---

## Abstract

The wallet SDK can now send Utility Registry tokens (pre-approval work currently in progress). Three practical problems remain once that lands: the SDK already lets a caller create, renew, and cancel a Canton Coin pre-approval with itself as provider, but there is no support for operating that choice at scale (bulk monitoring, retry-safe renewal); the existing unified transaction-history read path lacks per-registry symbol/metadata enrichment; and there is no standard way for one Canton wallet to request a payment from another. This proposal addresses all three as a single coherent scope, the lifecycle around a token transfer, not just the transfer itself.

---

## Specification

### 1. Objective

The work in progress on Utility Registry token pre-approvals closes the send gap. It does not close the gaps around sending. Default-provider (validator operator) Canton Coin pre-approvals already auto-renew via validator-app automation, and Utility Registry (CIP56) pre-approvals have no expiry at all, so neither of those needs renewal tooling. Wallets and exchanges that choose to be their own provider party on a Canton Coin pre-approval already have the primitives for it: the SDK's pre-approval namespace ships `create` (with a caller-supplied provider party), `renew`, `cancel`, and a `fetchStatus` poller. What's missing sits one layer up. `fetchStatus` checks one receiver party at a time, so there is no way to scan everything a caller manages for pre-approvals nearing expiry, and nothing retries a renewal that fails partway, and nothing operates that at the batch scale an exchange running many external parties needs. Separately, the SDK's `listHoldingTransactions` already returns a registry-agnostic transaction history in one call, but it doesn't carry human-readable per-registry instrument metadata (symbol, display name) alongside it, so a caller still has to fetch that separately per registry and join it by hand. And when a merchant or dApp wants to ask a user to send a specific token, there is no Canton equivalent of a payment URI, every app invents a format, so wallets cannot interoperate on incoming payment requests.

A prior draft of this milestone also scoped in tracking and redemption of the `AppRewardCoupon`s a non-default provider earns. That mechanism is being replaced by CIP-0104 (Approved), whose Increment 4 (targeted for MainNet in August 2026) removes `AppRewardCoupon` and `FeaturedAppActivityMarker` creation entirely and moves reward eligibility to parties holding an active `FeaturedAppRight` acting as transaction confirmers — being named provider on a pre-approval alone will no longer generate any reward. Since Featured App status requires a separate DSO governance vote and CC collateral disproportionate to this proposal's scope, reward-coupon tracking and redemption are dropped from this milestone rather than rebuilt against a mechanism being sunset.

These three gaps are a natural follow-on to the pre-approval work and are best handled together, because they share the same SDK surface area and the same integration tests.

### 2. Implementation Mechanics

**Non-default-provider pre-approval orchestration.** The SDK already supports creating, renewing, and cancelling a Canton Coin pre-approval with a caller-chosen provider party (`sdk.amulet.preapproval.command.create`, `.renew`, `.cancel`, `.fetchStatus`). This proposal builds the operational layer on top that wallets and exchanges actually need: a `getExpiringPreapprovals(withinDays)` helper that scans every pre-approval a caller manages, since `fetchStatus` today only checks one party at a time; scheduled renewal with retry/backoff so a transient failure does not silently drop a pre-approval; and a batch orchestration wrapper — concurrency control plus aggregated result reporting over the scan and renewal helpers — for exchanges managing many external parties at once, plus monitoring/alerting hooks so a wallet team can see expiry state without polling manually. No new protocol; no new contracts. This does not touch default-provider Canton Coin pre-approvals (already auto-renewed by validator-app automation, confirmed at 30 days out per the [Canton docs](https://github.com/canton-network/splice/blob/main/docs/src/background/preapprovals.rst#L85-L91)) or Utility Registry pre-approvals (no expiry, nothing to renew). That same doc section is explicit that non-default-provider preapprovals need their own renewal automation, which is the gap this milestone closes.

**Multi-registry transfer history enrichment.** `listHoldingTransactions` already reads at the ledger's standardized Token Standard interface level, so it returns a registry-agnostic transaction history in one call, with `instrumentId` per line item, amounts as fixed-point Daml decimals, and offset-based cursor pagination (`afterOffset`/`beforeOffset` in, `nextOffset` out) already built in — no per-registry decimal conversion or pagination work needed here. What it doesn't carry is human-readable metadata: each event's `instrumentId` is a raw `{admin, id}` pair, and there is no reverse lookup anywhere in the SDK from that `admin` party back to the registry URL needed to call the SDK's existing `getInstrumentById`/`listInstruments`/`registriesToAssets` metadata helpers. This proposal adds that resolution step plus a join-and-cache layer over the metadata those helpers already return (symbol, display name). A caller gets one enriched list regardless of how many registries a user holds tokens in, without resolving registries or hand-joining metadata calls themselves.

**Payment request format.** Define a Canton payment URI format and ship a parser and builder in the SDK. The party-identification component is a CAIP-10 account id (`canton:<network-id>:<party-id>`), the same format WalletConnect already uses for Canton account identifiers — Canton's networks are operator-defined with no fixed mainnet/testnet, so a bare party ID would be ambiguous across them. Payment-specific fields (registry ID, instrument ID, amount, and optional memo, expiry, nonce) are layered on as query params, the same pattern EIP-681 uses for Ethereum payment URIs on top of a chain-qualified address. Example: `canton:<network-id>:<party-id>?registry=REGISTRY_ID&instrument=INSTRUMENT_ID&amount=10.00&memo=invoice-42`. The builder accepts a token type, amount, and party and returns a valid URI. The parser returns a structured object that feeds directly into the transfer path. The spec is published as a short Canton Improvement Proposal, including a section on its relationship to CAIP-10 (account identification), CAIP-19 (asset identification — no Canton profile currently exists), and CAIP-358 (a chain-agnostic payment-request method built on JSON-RPC rather than a URI scheme — different transport, not a competing spec).

**Upstream contribution and reference integration.** All three pieces are contributed as pull requests to `canton-network/wallet`, matching the repository's code style and test conventions. A reference integration runs end to end against real registry endpoints: create a non-default-provider Canton Coin pre-approval, watch it approach expiry, renew it, send a Utility Registry token via a parsed payment request, and verify the transfer appears in the unified history. Kept green in CI.

### 3. Architectural Alignment

All three pieces extend the official wallet SDK rather than building a parallel library. None changes the protocol or existing SDK APIs. Non-default-provider pre-approval orchestration sits above the existing pre-approval primitives. Transfer history metadata enrichment sits above the existing, already-unified `listHoldingTransactions` read path. Payment request format introduces a new URI scheme but does not touch the ledger. Backward compatible in all three cases.

### 4. Backward Compatibility

Additive only. No changes to existing SDK APIs, on-ledger contracts, or runtime behavior.

---

## Milestones and Deliverables

### Milestone 1 — Non-Default-Provider Pre-Approval Orchestration
- **Duration:** Weeks 1–5
- **Deliverables:** Bulk monitoring and retry-safe renewal orchestration on top of the SDK's existing create/renew/cancel pre-approval primitives; `getExpiringPreapprovals(withinDays)` query helper; batch monitor/renew operations for exchanges; expiry monitoring/alerting hooks; unit tests; upstream pull request to `canton-network/wallet`.

| Task | Hours |
|---|---|
| Research non-default-provider setup and existing create/renew/cancel primitives | 6h |
| Implement `getExpiringPreapprovals(withinDays)` bulk-scan helper across all pre-approvals a caller manages | 8h |
| Implement scheduled renewal orchestration on top of the existing `renew()` call, with retry/backoff | 12h |
| Batch orchestration wrapper (concurrency + aggregated reporting) over the scan/renewal helpers, for exchanges managing many parties | 5h |
| Monitoring/alerting hooks (metrics/webhooks) for pre-approval expiry state | 7h |
| Unit tests and edge case coverage | 8h |
| Upstream PR preparation and review cycles | 9h |
| Documentation and inline examples | 5h |
| **Milestone 1 Total** | **60h** |

---

### Milestone 2 — Multi-Registry Transfer History Enrichment
- **Duration:** Weeks 5–11
- **Deliverables:** Admin→registryUrl resolution layer; join-and-cache layer attaching per-registry instrument metadata (symbol, display name) from the existing `getInstrumentById`/`listInstruments`/`registriesToAssets` helpers onto the existing, already-paginated `listHoldingTransactions` stream; upstream pull request.

| Task | Hours |
|---|---|
| Design join + cache layer over existing `getInstrumentById`/`listInstruments`/`registriesToAssets` metadata helpers | 7h |
| Admin→registryUrl resolution layer (no reverse lookup exists in the SDK from `instrumentId.admin` to a registry URL) | 7h |
| Implement metadata join + cache layer over the transaction stream (symbol, display name, instrument ID) | 28h |
| Integration tests against Canton Coin and Utility Registry | 10h |
| Upstream PR preparation and review cycles | 10h |
| Documentation and runnable examples | 5h |
| **Milestone 2 Total** | **67h** |

---

### Milestone 3 — Payment Request Format and Reference Integration
- **Duration:** Weeks 11–17
- **Deliverables:** Canton payment URI spec published as a CIP draft; parser and builder in the SDK; upstream pull request; reference integration running end to end in CI.

| Task | Hours |
|---|---|
| Draft Canton payment URI CIP spec, including relationship to CAIP-10, CAIP-19, and CAIP-358 | 18h |
| Implement URI builder | 10h |
| Implement URI parser with structured output | 15h |
| End-to-end reference integration (expiry → payment request → history) | 25h |
| CI setup and green-build maintenance | 10h |
| Outreach and onboarding of 2 independent wallet teams | 10h |
| Upstream PR preparation and review cycles | 10h |
| Documentation and integration guide | 5h |
| **Milestone 3 Total** | **103h** |

---

### Milestone 4a — External Auditor Fee (Pass-Through)
- **Duration:** Weeks 17–20
- **Deliverables:** Engagement of an independent third-party security firm to audit all SDK additions; published audit report.

This is a direct pass-through to the external auditor. Estimated cost: USD 25,000–30,000, billed in CC at spot rate at time of engagement. This line is not development time — it is the auditor's fee paid to a firm outside this proposal.

### Milestone 4b — Developer Remediation
- **Duration:** Weeks 17–20 (concurrent with audit)
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
| 1 | M1 | Research existing pre-approval primitives |
| 2 | M1 | Implement `getExpiringPreapprovals` bulk-scan helper and renewal orchestration with retry/backoff |
| 3 | M1 | Bulk operations for exchanges (batch monitor/renew) |
| 4 | M1 | Monitoring/alerting hooks (expiry state) |
| 5 | M1 | Unit tests, upstream PR and docs — **M1 delivery** |
| 5 | M2 | Design join + cache layer over existing metadata helpers |
| 6 | M2 | Admin→registryUrl resolution layer |
| 7–8 | M2 | Implement metadata join + cache layer |
| 9 | M2 | Integration tests |
| 10–11 | M2 | Upstream PR and docs — **M2 delivery** |
| 11–12 | M3 | CIP spec draft and URI builder |
| 13 | M3 | URI parser implementation |
| 14–15 | M3 | End-to-end reference integration and CI setup |
| 16 | M3 | Wallet team outreach and upstream PR |
| 17 | M3 | Docs and final review — **M3 delivery** |
| 17 | M4 | Audit scope prep and auditor briefing |
| 18–19 | M4 | Audit execution and remediation |
| 20 | M4 | Re-test, final report publication — **M4 delivery** |

**Total project duration:** 20 weeks
**Total build hours (M1–M3):** 230h at 2,000 CC/h
**Total remediation hours (M4b):** 40h at 2,000 CC/h

---

## Acceptance Criteria

- Non-default-provider pre-approval monitoring/renewal orchestration for external parties merged into `canton-network/wallet`.
- Multi-registry transfer history metadata-enrichment layer merged into `canton-network/wallet`.
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
| M1 — Non-default-provider pre-approval orchestration | SDK helpers + upstream PR | 60h | 2,000 CC/h | 120,000 CC |
| M2 — Multi-registry transfer history enrichment | Metadata enrichment + upstream PR | 67h | 2,000 CC/h | 134,000 CC |
| M3 — Payment request format | CIP + SDK + reference integration | 103h | 2,000 CC/h | 206,000 CC |
| **Build Total** | | **230h** | | **460,000 CC** |

### Audit Budget (M4 — additional, on top of build)

| Line | Description | CC |
|---|---|---|
| M4a — External auditor fee | Pass-through to independent security firm | USD 25,000–30,000 in CC at spot rate |
| M4b — Developer remediation | 40h × 2,000 CC/h | 80,000 CC |
| **Audit Total** | | **80,000 CC + auditor fee** |

**Total Funding Request: 540,000 CC + auditor pass-through**

Payment for each milestone is triggered upon committee acceptance of that milestone's deliverables.

### Note on Audit Cost
The Canton Foundation has covered external audit costs for wallet-critical proposals in the past and is welcome to do so here as well. If the Foundation prefers to fund M4a (the auditor fee) directly rather than as a pass-through, the build budget of 460,000 CC and developer remediation of 80,000 CC remain unchanged.

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

The wallet team is building Utility Registry token pre-approvals now. The moment that lands, three gaps become the next real problem for wallet teams: any wallet or exchange choosing to run its own provider party on a Canton Coin pre-approval has no SDK support for operating that choice at scale, transaction history lacks per-registry metadata enrichment, and there is no interoperability on payment requests. Each gap has been raised in community channels by builders hitting it. Addressing them as a package means the ecosystem gets payment lifecycle completeness once rather than each wallet team solving each piece separately.

---

## Rationale

The three pieces share the same SDK surface and the same integration tests, so bundling them is more efficient than three separate proposals and easier for the committee to evaluate as a coherent scope. Each piece is independently useful but they are most valuable together — a wallet that can send any token, operate its own pre-approval provider relationship at scale, show an enriched transaction history, and accept incoming payment requests has a complete feature set. The payment request format in particular has the most ecosystem leverage because it enables interoperability across any wallet that implements the transfer path, including wallets the committee has no direct visibility into.
