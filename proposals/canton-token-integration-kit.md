## Development Fund Proposal

**Author:** Ayush Singh (Individual contributor, ayushsinghmi711@gmail.com)
**Status:** Submitted
**Created:** 2026-05-20
**Label:** token-asset-standards
**Champion:** need Champion

---

## Abstract

This proposal funds Utility Registry token support for the official Canton wallet SDK, plus the CIP-56 integration documentation that the ecosystem is missing today. The wallet SDK already supports transfer, allocation, and utxos, and it supports transfer pre-approval for Canton Coin. It does not yet handle generic transfer pre-approval for Utility Registry tokens, specifically the factory resolution (default factory versus pre-approval factory) and the shared choice-context and disclosed-contract handling for non-Amulet registries. This work adds that support upstream, with a tested reference integration against real registries and completed CIP-56 docs, so any team can support any token through one path.

---

## Specification

### 1. Objective

Even with the wallet SDK in place, a team integrating a real Utility Registry token still has to solve the hardest parts by hand. The SDK has pre-approval handling for Canton Coin in `amulet/preapproval.ts`, but the token namespace has no generic pre-approval path for Utility Registry tokens, and the factory resolution that the token standard requires is not present. A team also has to assemble choice context and disclosed contracts for non-Amulet registries on its own. This is the gap that the Rapid Chain USDCx write-up and the follow-up on the global-synchronizer mailing list described.

The objective is one focused deliverable: add Utility Registry token support to the official wallet SDK and complete the CIP-56 docs, so a team integrates any token with the same path it already uses for Canton Coin.

### 2. Implementation Mechanics

The work has the following parts.

- **Utility token pre-approval and factory resolution.** Add a pre-approval path for Utility Registry tokens to the SDK token namespace. It detects whether the receiver has a pre-approval and resolves the correct factory: the pre-approval factory when one exists, and the default factory otherwise, as the token standard defines.
- **Choice context and disclosed contracts for non-Amulet registries.** Add shared handling so transfer and allocation assemble the choice context and disclosed contracts a registry requires, using one path for Canton Coin and Utility Registry tokens.
- **Upstream contribution.** Contribute these as pull requests to the `canton-network/wallet` repository, matching its code style, with unit tests and the repository's documentation snippet tests.
- **Reference integration.** Build a tested integration that runs an end-to-end transfer of a Utility Registry token and of Canton Coin against real registry endpoints, kept green in CI, so teams can copy a known-good path.
- **CIP-56 documentation.** Complete the preliminary CIP-56 docs with runnable, tested examples covering registry metadata, holdings, transfer instruction, choice context, disclosed contracts, and pre-approval factory resolution.

### 3. Architectural Alignment

The work extends the official wallet SDK and the CIP-56 docs instead of building a parallel library. It uses the registry APIs and on-ledger contracts the token standard defines, and it does not change the protocol. It follows the App Building and Developer Experience priority by lowering integration friction for wallets and applications, and it aligns with CIP-56 and the V2 token standard work in CIP-112.

### 4. Backward Compatibility

No backward compatibility impact. The work is additive to the wallet SDK. It does not change existing SDK APIs, ledger contracts, or runtime behavior.

---

## Milestones and Deliverables

### Milestone 1: Utility token pre-approval and factory resolution
- **Estimated Delivery:** 7 weeks from start
- **Focus:** Close the pre-approval gap for Utility Registry tokens.
- **Deliverables / Value Metrics:** Pre-approval path and factory resolution in the SDK token namespace; unit tests; upstream pull request to `canton-network/wallet`. A team can send a Utility Registry token to a receiver whether or not a pre-approval exists.

### Milestone 2: Non-Amulet choice context and reference integration
- **Estimated Delivery:** 13 weeks from start
- **Focus:** One path for Canton Coin and Utility Registry tokens, proven against real registries.
- **Deliverables / Value Metrics:** Shared choice-context and disclosed-contract handling; a reference integration that transfers a Utility Registry token and Canton Coin end to end against real registry endpoints, kept green in CI. A team can follow a known-good path for any token.

### Milestone 3: CIP-56 documentation and adoption
- **Estimated Delivery:** 18 weeks from start
- **Focus:** Documentation and real usage.
- **Deliverables / Value Metrics:** Completed CIP-56 docs with runnable, tested examples; onboarding support for teams. Measured by independent teams that integrate a token using the SDK path and the docs.

---

## Acceptance Criteria

The Tech & Ops Committee will evaluate completion based on ecosystem value:

- The Utility token pre-approval, factory resolution, and choice-context contributions are merged into `canton-network/wallet`.
- The reference integration transfers a Utility Registry token and Canton Coin end to end against real registry endpoints on TestNet or MainNet.
- The CIP-56 docs and examples are published, and at least 3 independent teams integrate a token using this path.

Acceptance is based on real adoption, not on the delivery of an artifact alone.

---

## Funding

**Total Funding Request:** 550,000 CC

### Payment Breakdown by Milestone
- Milestone 1 (Utility token pre-approval and factory resolution): 250,000 CC upon committee acceptance
- Milestone 2 (Non-Amulet choice context and reference integration): 175,000 CC upon committee acceptance
- Milestone 3 (CIP-56 documentation and adoption): 125,000 CC upon committee acceptance of adoption evidence

### Volatility Stipulation
The project duration is under 6 months. Should the timeline extend beyond 6 months due to Committee-requested scope changes, the remaining milestones will be renegotiated to account for USD and CC price movement.

---

## Adoption and Go-to-Market

- The SDK changes land upstream, so every team that uses the wallet SDK gets Utility token support by default.
- The reference integration and the completed docs give a new team a known-good path to follow.
- Direct outreach to wallet and application teams that have raised token-integration issues on the mailing list and in community channels.

---

## Maintenance and Sustainability

The SDK contributions are maintained in the official `canton-network/wallet` repository under its existing process, so they continue with the SDK after the grant. Ayush Singh maintains the reference integration and the documentation examples, and tracks token-standard changes including the V2 token standard.

---

## Co-Marketing

Upon release, the team will work with the Foundation on an announcement, a technical blog post, and a developer session for wallet and application builders.

---

## Motivation

Wallets and applications that support third-party tokens need correct Utility Registry token handling, and the pre-approval and choice-context parts are where teams get stuck today. Rapid Chain published a write-up of the issues hit while integrating USDCx, and the follow-up on the global-synchronizer mailing list, including notes from Digital Asset and a transfer pre-approval clarification from Sergei Ganebnyi, confirmed these points affect many builders. Adding the support upstream means the whole ecosystem gets it once, instead of each team solving it again.

---

## Rationale

The right approach is to extend the official wallet SDK rather than build a parallel library, which also follows the guidance to extend what exists. The SDK already covers the common path; this work fills the specific Utility Registry token gap and proves it against real registries. It complements the local testing harness work in proposal #335 and the SDK itself, and it keeps one path for Canton Coin and Utility Registry tokens so teams have less to learn and less to maintain.
