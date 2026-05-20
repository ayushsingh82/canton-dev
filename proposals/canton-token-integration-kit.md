## Development Fund Proposal

**Author:** Ayush Singh (Individual contributor, ayushsinghmi711@gmail.com)
**Status:** Submitted
**Created:** 2026-05-20
**Label:** token-asset-standards
**Champion:** need Champion

---

## Abstract

This proposal funds an open-source client library and conformance test suite that lets any Canton wallet or application support token-standard assets through one code path. It covers Canton Coin and Utility Registry tokens such as USDCx, and handles the registry APIs, choice context, disclosed contracts, transfer instructions, and transfer pre-approval behavior that wallet teams currently implement by hand. The conformance suite gives a team an objective pass or fail check that their token support is correct before they go live on MainNet.

---

## Specification

### 1. Objective

Integrating CIP-56 token-standard assets into a wallet or application is error-prone and under-documented. Teams repeatedly hit the same problems: missing Utility Registry packages, the difference between Canton Coin flows and Utility Registry token flows, calling `TransferInstruction_Accept` without the correct context, assembling `choiceContextData` and `disclosedContracts`, and resolving transfer pre-approvals. A visible balance is not the same as correct token support, and these gaps are usually found late, during testing against a live registry.

The objective is one focused deliverable: a tested client library plus a conformance test suite so that a team can support any token-standard asset correctly with a single integration, and can prove that support before deployment.

### 2. Implementation Mechanics

The kit has the following parts.

- **Typed client library.** A client (TypeScript first) that wraps the token-standard registry APIs: metadata and instruments, holdings, transfer instruction, allocation, and allocation instruction. It builds the on-ledger exercises a wallet needs without the team writing low-level plumbing.
- **Choice-context and disclosed-contracts resolver.** Given a transfer request, it queries the registry choice context, collects the disclosed contracts, and assembles a valid exercise so that `TransferInstruction_Accept` and related choices are called with the context they require.
- **Transfer pre-approval handling.** It detects whether the receiver has a pre-approval. For Canton Coin it reads the `transfer-preapproval` contract id returned in the choice context. For Utility Registry tokens it selects the correct factory id, using the pre-approval factory when one exists and the default factory otherwise.
- **Registry adapters.** Canton Coin is read through an SV Scan endpoint, for example `/registry/metadata/v1/instruments`. Utility Registry tokens are read through their own registry API. The adapters share one interface so an application uses the same calls for both.
- **Conformance test suite.** A runnable set of tests that a wallet or application executes against DevNet or TestNet to confirm it lists balances correctly, builds and accepts transfers, and handles both pre-approval and non-pre-approval receivers. The suite produces a pass or fail report.
- **Reference sample application.** A small open-source app that shows an end-to-end transfer of Canton Coin and of one Utility Registry token, so teams can copy a known-good integration.

### 3. Architectural Alignment

The kit aligns with the CIP-56 token standard and with the V2 token standard work in CIP-112. It uses the registry APIs and on-ledger contracts the standard already defines, and it does not change the protocol or any ledger contract. It follows the App Building and Developer Experience priority by lowering integration friction for wallets and applications. It works alongside the Canton Network dApp SDK and the published token-standard documentation, and it can adopt the upcoming CN Credential standard for resolving admin party ids to off-ledger registry endpoints when that lands.

### 4. Backward Compatibility

No backward compatibility impact. The kit is an additive client library and test suite. It does not modify existing ledger contracts, registries, or runtime behavior.

---

## Milestones and Deliverables

### Milestone 1: Core library and registry adapters
- **Estimated Delivery:** 6 weeks from start
- **Focus:** One code path for reading balances and building transfers across Canton Coin and Utility Registry tokens.
- **Deliverables / Value Metrics:** Typed client library; Canton Coin and Utility Registry adapters; transfer build and accept; developer documentation. A wallet can list balances and transfer Canton Coin and one Utility Registry token using the same calls.

### Milestone 2: Pre-approval handling and conformance suite
- **Estimated Delivery:** 12 weeks from start
- **Focus:** Correctness verification before MainNet.
- **Deliverables / Value Metrics:** Pre-approval detection and factory resolution; runnable conformance test suite with pass or fail report; CI integration. A team can verify correct token support before deployment.

### Milestone 3: Reference app and adoption
- **Estimated Delivery:** 18 weeks from start
- **Focus:** Real usage by other teams.
- **Deliverables / Value Metrics:** Open-source reference application; integration guide; onboarding support for adopting teams. Measured by independent teams that integrate the kit and pass the conformance suite.

---

## Acceptance Criteria

The Tech & Ops Committee will evaluate completion based on ecosystem value:

- At least 3 independent wallets or applications integrate the kit and pass the conformance suite against MainNet token registries.
- The reference application demonstrates an end-to-end transfer of Canton Coin and of one Utility Registry token, including a pre-approval and a non-pre-approval receiver.
- Documentation and the conformance report format are published and open-source under Apache-2.0.

Acceptance is based on real adoption, not on the delivery of an artifact alone.

---

## Funding

**Total Funding Request:** 600,000 CC

### Payment Breakdown by Milestone
- Milestone 1 (Core library and adapters): 250,000 CC upon committee acceptance
- Milestone 2 (Pre-approval and conformance suite): 200,000 CC upon committee acceptance
- Milestone 3 (Reference app and adoption): 150,000 CC upon committee acceptance of adoption evidence

### Volatility Stipulation
The project duration is under 6 months. Should the timeline extend beyond 6 months due to Committee-requested scope changes, the remaining milestones will be renegotiated to account for USD and CC price movement.

---

## Adoption and Go-to-Market

- Direct outreach to wallet and application teams that have raised token-integration issues on the mailing list and in community channels.
- An integration guide and the reference app published openly, so a new team can follow a known-good path.
- A short conformance badge that a wallet can show once it passes the suite, giving users a signal of correct token support.

---

## Maintenance and Sustainability

The library and conformance suite are released under Apache-2.0 and hosted in a public repository. After the grant, Ayush Singh maintains the kit and tracks token-standard changes, including the V2 token standard, and accepts community contributions. A small maintenance milestone can be proposed separately once the standard stabilizes.

---

## Co-Marketing

Upon release, the team will work with the Foundation on an announcement, a technical blog post or case study, and a developer session for wallet and application builders.

---

## Motivation

Wallets and applications that hold assets need correct token-standard support, and integration is a known blocker. A team at Rapid Chain published a public write-up of the issues they hit while integrating USDCx, and the follow-up discussion on the global-synchronizer mailing list, including notes from Digital Asset, confirmed that several of these points affect many builders. Most wallets and applications that handle assets will need this support as more token issuers arrive, so a shared, tested kit removes repeated work across the ecosystem.

---

## Rationale

The right approach is to extend the existing token standard rather than build a parallel path. One tested code path that covers both Canton Coin and Utility Registry tokens reduces the chance of subtle errors, and the conformance suite turns correctness into an objective check rather than a manual review. The kit complements the Canton Network dApp SDK and the existing documentation, and it reuses the registry APIs the standard already defines, so it fits the ecosystem instead of replacing parts of it.
