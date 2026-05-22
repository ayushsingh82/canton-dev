## Development Fund Proposal

**Author:** Ayush Singh (Individual contributor, ayushsinghmi711@gmail.com)
**Status:** Submitted
**Created:** 2026-05-20
**Label:** daml-tooling
**Champion:** need Champion

---

## Abstract

This proposal funds an open-source static analysis tool that compares two compiled Daml package versions and reports whether the new version is a safe smart contract upgrade of the old one. When it finds a breaking change, it points to the exact template, field, or choice and the source location. The tool runs locally and in CI, so teams catch upgrade problems before they deploy to Canton rather than during a failed upgrade.

---

## Specification

### 1. Objective

Daml smart contract upgrades follow rules about what may change between two versions of a package. For example, new record fields must be optional and added at the end, field types must stay the same, and removed templates or choices break compatibility. Today many teams check these rules by reading code or by attempting an upgrade and watching it fail. Both approaches are slow and find problems late.

The objective is one focused deliverable: a tool that statically verifies upgrade compatibility between two Daml package versions and reports each breaking change with its location, so an upgrade is checked before deployment.

### 2. Implementation Mechanics

The tool takes two compiled DAR files for the same package, an older version and a newer version. It then does the following.

1. Reads the Daml-LF from each DAR and builds the set of templates, choices, records, variants, enums, and interfaces.
2. Matches these elements across the two versions by name.
3. Applies the documented upgrade compatibility rules. Examples include: new fields must be optional and added at the end, field types must match, removed fields are flagged, removed templates or choices are flagged, and changes to signatories or observers are flagged.
4. Reports each incompatibility with the template or field name, the rule that was broken, and the source file and line where available.
5. Produces a JSON report for other tools and a readable summary for a person.
6. Returns a non-zero exit code when a breaking change is found, so it can gate a CI pipeline.

The tool can also check a whole DAR dependency tree to a chosen depth, so a project can verify its dependencies as well as its main package.

### 3. Architectural Alignment

The tool operates on the same compiled artifact that is deployed to Canton, and the analysis is static, with no execution of contracts. It aligns with the Daml smart contract upgrade feature and fits naturally into existing build and CI workflows next to `daml build` and package tooling such as dpm. It follows the App Building and Developer Experience priority and the Stability and Maintainability priority by making network and package upgrades safer and lower cost. It follows the design of the already approved Daml Package Analyzer, which also reads compiled packages and reports source-level findings.

### 4. Backward Compatibility

No backward compatibility impact. The tool is a read-only analyzer. It does not change Daml applications, packages, or runtime behavior.

---

## Milestones and Deliverables

### Milestone 1: Core comparison engine
- **Estimated Delivery:** 6 weeks from start
- **Focus:** Detect the most common breaking changes in templates and records.
- **Deliverables / Value Metrics:** DAR parsing; element matching; rule checks for record and template field changes; JSON report and readable summary. A team can compare two package versions and see field-level breaking changes.

### Milestone 2: Full rule coverage and CI mode
- **Estimated Delivery:** 12 weeks from start
- **Focus:** Complete the rule set and make it usable in pipelines.
- **Deliverables / Value Metrics:** Rule checks for choices, interfaces, variants, enums, and signatory or observer changes; source location mapping; non-zero exit on breaking changes; a ready-to-use GitHub Action. A team can gate deployments on upgrade safety.

### Milestone 3: Dependency-tree analysis and adoption
- **Estimated Delivery:** 18 weeks from start
- **Focus:** Real usage in other projects.
- **Deliverables / Value Metrics:** Dependency-tree analysis to a chosen depth; a public test corpus of breaking and non-breaking changes; onboarding support. Measured by independent Daml projects that adopt the tool in CI.

---

## Acceptance Criteria

The Tech & Ops Committee will evaluate completion based on ecosystem value:

- At least 3 independent Daml projects adopt the tool in their CI to gate upgrades.
- The tool correctly classifies a published test corpus of known breaking and non-breaking changes.
- Documentation and the report format are published and open-source under Apache-2.0.

Acceptance is based on real adoption, not on the delivery of an artifact alone.

---

## Funding

**Total Funding Request:** 450,000 CC

### Payment Breakdown by Milestone
- Milestone 1 (Core comparison engine): 200,000 CC upon committee acceptance
- Milestone 2 (Full rule coverage and CI mode): 150,000 CC upon committee acceptance
- Milestone 3 (Dependency-tree analysis and adoption): 100,000 CC upon committee acceptance of adoption evidence

### Volatility Stipulation
The project duration is under 6 months. Should the timeline extend beyond 6 months due to Committee-requested scope changes, the remaining milestones will be renegotiated to account for USD and CC price movement.

---

## Adoption and Go-to-Market

- A ready-to-use GitHub Action so a project can add the check in a few lines.
- An integration guide and a public test corpus, so teams trust the results and can contribute new cases.
- Outreach to teams that ship Daml packages and to node operators who coordinate upgrades.

---

## Maintenance and Sustainability

The tool is released under Apache-2.0 in a public repository. After the grant, Ayush Singh maintains it and tracks changes to the Daml smart contract upgrade rules as the language evolves, and accepts community contributions. A separate maintenance milestone can be proposed if ongoing upkeep is needed.

---

## Co-Marketing

Upon release, the team will work with the Foundation on an announcement, a short technical blog post, and a developer session showing the tool in a real upgrade workflow.

---

## Motivation

Package upgrades are a recurring source of risk for teams building on Canton, and a broken upgrade can be costly to diagnose after the fact. As the network runs different releases across DevNet, TestNet, and MainNet, and as more applications ship new package versions, a static check that catches upgrade breakages before deployment saves time for every team that ships Daml code. The benefit grows with the number of applications on the network.

---

## Rationale

The right approach is a static tool that reads the compiled package, because the deployed artifact is exactly what an upgrade must stay compatible with, and static analysis gives a fast answer with no test ledger required. This proposal extends the existing tooling pattern rather than replacing it: the already approved Daml Package Analyzer reads one package to report cross-package authority, while this tool compares two versions of one package to report upgrade safety. The two are complementary, and sharing the same artifact and report style keeps them easy to use together.
