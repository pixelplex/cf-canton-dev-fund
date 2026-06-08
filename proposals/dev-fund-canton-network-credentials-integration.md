# Development Fund Proposal

## Canton Network Credentials — Ecosystem Integration

**Author:** Vlad Kokosh ([v.kokosh@pixelplex.io](mailto:v.kokosh@pixelplex.io))  
**Organization:** PixelPlex  
**Website:** <https://ccview.io>  
**CIP Reference:** <https://github.com/canton-foundation/cips/pull/204>  
**Related Implementation:** <https://github.com/hyperledger-labs/splice/pull/3416>  
**Tech & Ops Champion:** *[To be confirmed]*  
**Status:** Submission Draft  
**Created:** 2026-05-14  
**Label:** canton-apis

---

## Abstract

This proposal requests Development Fund support to deliver end-to-end runtime integration for the Canton Network Credentials standard. The work is focused on upstream Splice and ecosystem SDK integration required for production adoption.

The implementation scope is organized around the full credentials delivery chain in Splice:

- strengthening and upstreaming the Daml credentials implementation;
- adding runtime support for credential lifecycle automation in Super Validator services;
- exposing credential API surfaces through Scan;
- enabling the corresponding HTTP API routing through validator proxy layers;
- standardizing registry API discovery by publishing off-ledger endpoints via credentials metadata for both the CC token registry and the DSO-run credentials registry.

Alongside runtime delivery, the proposal includes upstream SDK integration in `canton-network/wallet` (credentials support within the existing SDK architecture), plus conformance vectors and integrator documentation.

The intended outcome is that the CIP's functionality is available through a production runtime path, with merged upstream changes and reproducible end-to-end behavior.
The grant is structured to take the credentials standard from upstream implementation to production-ready ecosystem integration.

---

## About the Proposer

PixelPlex is a General Partner of the Canton Foundation and an active contributor to the Canton ecosystem. The team has delivered and operates several production systems that are in daily use across the network:

- **CCView Block Explorer** ([ccview.io](https://ccview.io)) — a Canton Network explorer widely used across the ecosystem, including by the Canton Foundation itself. CCView’s API layer serves data to top parties including Canton Foundation, Modulo Finance, CBTC, Circle, and others. CCView is also one of the primary consumers of credential data on the network.
- **Console Wallet** — one of the largest wallet products on Canton, supporting EVM compatibility, bridges, swaps, and other features across browser extension and mobile apps. Console Wallet is one of the primary consumers of party / registry metadata.
- **Governance contributions** — PixelPlex co-authored the dApp CIP, authored the Party Profile Credentials CIP ([#169](https://github.com/canton-foundation/cips/pull/169)), and is actively engaged in the review of the Canton Network Credentials Standard ([#204](https://github.com/canton-foundation/cips/pull/204)).

-----

## Ecosystem Demand

The Canton Network Credentials Standard ([#204](https://github.com/canton-foundation/cips/pull/204)) is a foundational primitive for service discovery, profile publication, name resolution, and verified identity flows. Once it ships, every wallet, explorer, and dApp on the network will need to interact with it — both on-ledger (Daml workflows) and off-ledger (HTTP API).

Today the on-ledger implementation is being actively built by the Splice team, while the off-ledger consumer layer still requires coordinated upstream SDK and runtime delivery for broad ecosystem adoption. Even on the on-ledger side, infrastructure of this kind benefits from additional engineering capacity working alongside the core team.

PixelPlex is already engaged in the PR #204 review process and operates two of the primary applications (CCView and Console Wallet) that will adopt this standard.
The intended scope of Daml contributions has been discussed with the Splice maintainers, who have indicated upstream contributions of this kind are welcome. Funding the full integration layer once, as a public good, lets the entire ecosystem absorb the standard faster and more consistently than it would otherwise.

-----

## Problem Statement

Foundational standards deliver value only when the surrounding integration layer is in place. The Canton Network Credentials Standard currently has:

- a draft CIP under review;
- a draft on-ledger implementation in Splice;
- no unified runtime integration path across the app/API/proxy surfaces;
- no shared conformance corpus that defines expected integration behavior;
- no integrator-facing delivery guide beyond the spec text.

The two possible outcomes are:

1. **No grant.** Each ecosystem team assembles its own integration path, derives behavior from partial references, runs ad-hoc validation, and depends on fragmented implementation patterns. Coverage and consistency depend on the bandwidth of individual teams.
2. **This grant.** A reusable, upstream integration path across Splice runtime components and SDK surfaces, validated by shared conformance and supported by implementation documentation.

This proposal funds outcome (2): the smallest reusable set of deliverables that enables confident ecosystem adoption on both on-ledger and off-ledger surfaces.

---

## Objective and Ecosystem Value

### Objective

Deliver a production-ready credentials integration path by upstreaming the required runtime and SDK changes, and by validating those changes through merged deliverables and reproducible end-to-end checks across the intended integration surfaces.

### Ecosystem Value

- **End-to-end delivery over partial artifacts.** The grant targets a complete integration path instead of disconnected implementation pieces.
- **Lower integration risk.** A shared upstream SDK path and explicit discovery conventions reduce ambiguity for teams integrating credentials.
- **Faster adoption across application teams.** Runtime, API, and proxy coverage are delivered together, so wallets, explorers, and dApps can adopt on a common baseline.
- **Reduced duplicated engineering effort.** Shared upstream components decrease repeated implementation and maintenance work across teams.
- **Durable public-good output.** Deliverables are contributed upstream under Apache 2.0 to remain reusable for the broader ecosystem.

---

## Scope

### In Scope

- Daml engineering contributions to credentials implementation in Splice.
- SV app changes to automate credential expiry handling.
- Scan app changes to serve credentials APIs, including registry info, lookup, and bulk retrieval surfaces.
- Validator proxy support for credentials HTTP APIs, including BFT-style reads across Scan backends.
- SV app support to publish off-ledger registry API URLs via credentials metadata:
  - CC token registry API URLs
  - DSO-run credentials registry API URLs
- Support for publishing and consuming issuer application URL discovery metadata (`credential-issuer-app-url`) in the standard credentials flow.
- Upstream SDK integration in `canton-network/wallet` (credentials support in existing SDK architecture; no standalone external reference client as a primary deliverable).
- Conformance vectors and compatibility validation.
- Integrator documentation and rollout guidance.

### Out of Scope

- Changes to Canton protocol consensus rules.
- Operating hosted registry infrastructure as a managed service.
- Independent third-party security audit (separate engagement if requested).

---

## Technical Approach

The work is organized into three tracks converging on production delivery outcomes.

### Track 1 — Splice Core Delivery

Upstream contributions in Splice covering:

- credentials Daml implementation refinements and tests;
- SV app automation for credential expiry;
- Scan app credentials API serving path (registry info, lookup, and bulk retrieval);
- validator proxy support for credentials HTTP APIs with BFT-read behavior;
- SV app publication of off-ledger registry URLs via credentials metadata;
- publication and usage path for issuer application URL discovery metadata.

### Track 2 — Upstream SDK Integration

Credentials support is integrated upstream in `canton-network/wallet`, following existing SDK architecture (namespace/module extension in `wallet-sdk`), so integrators adopt one maintained SDK path.

### Track 3 — Conformance and Documentation

- Versioned conformance vectors for credential flows;
- compatibility checks against target Splice builds;
- Integrator's Guide for operational integration and migration notes.

### Governance Path

Some Daml and runtime changes may require CIP alignment before they can be merged upstream. To keep delivery predictable, this proposal handles both cases explicitly:

- **Path A (no additional CIP required):** changes are delivered through the normal upstream PR and merge flow.
- **Path B (additional CIP/update required):** CIP work is treated as a formal dependency in the milestone plan, and milestone completion is assessed against both governance progress and merged technical deliverables available at that stage.

---

## Milestones

### Milestone 1 — Daml and Core Runtime Delivery

**Funding requested:** 400,000 CC  
**Estimated delivery:** ~2 months from kickoff

#### Deliverables

- Merged upstream Daml credentials implementation changes in Splice with test coverage.
- SV app support for automated credential expiry flow.
- Scan app serving of registry info and lookup credential APIs.
- Validator proxy support for credentials HTTP APIs with multi-scan read validation.
- Conformance vectors v0 (10+ scenarios) with CI validation.
- Integrator's Guide v0 covering architecture and primary runtime flows.

---

### Milestone 2 — API Discovery and SDK Delivery

**Funding requested:** 470,000 CC  
**Estimated delivery:** ~4 months from kickoff

#### Deliverables

- Upstream SDK implementation in `canton-network/wallet` covering primary credentials read/usage flows.
- Scan-side bulk retrieval API support for explorer ingestion use cases.
- SV app URL publication support for:
  - CC token registry API discovery
  - DSO-run credentials registry API discovery
- Issuer application URL discovery (`credential-issuer-app-url`) wired for wallet/app usage flows.
- Merged upstream changes across Daml/SV/Scan/proxy for end-to-end credentials flows.
- Conformance vectors at 20+ scenarios.
- Compatibility report against target Splice build.
- Integrator's Guide v0.x with integration recipes.

---

### Milestone 3 — Stabilization and Release Documentation

**Funding requested:** 330,000 CC  
**Estimated delivery:** ~5 months from kickoff

#### Deliverables

- Merged upstream set for Daml/SV/Scan/proxy credentials delivery path.
- Upstream SDK credentials integration in `canton-network/wallet`.
- Final conformance and compatibility matrix.
- Integrator's Guide v1.0.
- Delivery summary mapping each agreed scope item to merged PRs and runtime verification evidence.

---

## Funding Summary

| Milestone | Description | Funding (CC) |
|-----------|-------------|--------------|
| 1 | Daml and Core Runtime Delivery | 400,000 |
| 2 | API Discovery and SDK Delivery | 470,000 |
| 3 | Stabilization and Release Documentation | 330,000 |
| **Total** | | **1,200,000** |

Funding is sized based on internal estimation of approximately 1,650 engineering hours across Daml/runtime integration, SDK contributions, conformance, documentation, and production-ready completion work.

---

## Volatility Stipulation

This proposal is expected to complete within ~5 months. Milestone funding is denominated in fixed Canton Coin, and the recipient carries upside and volatility risk.

If delivery extends beyond that window due to committee-requested scope changes or material dependencies outside the proposer's control (including upstream governance/CIP timing), remaining milestones should be reviewed with the Tech & Ops Committee to account for significant CC/USD volatility.

---

## Delivery Timeline

| Phase | Estimated Timing |
|-------|------------------|
| Milestone 1 — Daml and Core Runtime Delivery | ~2 months from kickoff |
| Milestone 2 — API Discovery and SDK Delivery | ~4 months from kickoff |
| Milestone 3 — Stabilization and Release Documentation | ~5 months from kickoff |

---

## Open Source and Public Good

All deliverables are contributed upstream or published under permissive open-source licensing (Apache 2.0), with runtime changes primarily targeted at Splice and SDK changes targeted at `canton-network/wallet`.

This creates reusable ecosystem infrastructure and avoids fragmentation from parallel, standalone client implementations.

---

## Dependencies and Assumptions

This proposal assumes:

- The Canton Network Credentials standard ([#204](https://github.com/canton-foundation/cips/pull/204)) continues to progress toward acceptance.
- Splice maintainers continue accepting upstream contributions in Daml/runtime/API/proxy layers.
- `canton-network/wallet` maintainers continue accepting upstream SDK contributions for credentials support.
- Where additional governance/CIP steps are required for specific changes, those are tracked as milestone dependencies.

---

## Backward Compatibility

The work is additive at application/runtime level. Existing apps that do not adopt credentials flows continue operating unchanged.

---

## Security Considerations

This proposal does not introduce custody of user keys or wallet signing services. Runtime/API changes follow upstream review and existing security controls in Splice components.

Integrator documentation includes trust-boundary guidance for credential API consumption and discovery data handling.

---

## Risks and Mitigations

| Risk | Likelihood | Impact | Mitigation |
|------|------------|--------|------------|
| Upstream integration delays for runtime components | Medium | High | Milestones are sequenced by component; progress tracked via merged PR set per component |
| Governance/CIP dependency for subset of changes | Medium | Medium | Explicit governance path (A/B) in milestones; dependency status reported in each milestone summary |
| Divergence between SDK and runtime behavior | Medium | Medium | Conformance vectors + compatibility checks tied to upstream builds |
| Scope drift beyond agreed delivery items | Low | High | Scope constrained to agreed runtime/API/proxy/SDK items and validated per milestone |

---

## Co-Marketing and Ecosystem Contribution

Upon release, PixelPlex will collaborate with the Canton Foundation on:

- coordinated publication of merged delivery outputs;
- a technical write-up on integrating credentials flows through the runtime path;
- a developer workshop/session if requested by the Foundation.

---

## Maintenance and Sustainability

Within the grant period, PixelPlex maintains contributed integration artifacts (conformance vectors, docs, compatibility reports) and follows up on merged upstream contributions as needed.

Long-term maintenance of accepted runtime/SDK code follows the respective upstream maintainer models in Splice and `canton-network/wallet`.

---

## Rationale

**Why end-to-end runtime delivery.**  
Specifications and isolated artifacts are not enough for ecosystem adoption on their own. Delivering the runtime path across Daml, Scan, validator proxy, discovery metadata, and SDK integration provides one coherent implementation route for wallets, explorers, and dApps.

**Why include Daml work alongside runtime/API delivery.**  
The Splice team owns the core Daml implementation. This proposal adds parallel engineering capacity to upstream that work while also delivering the required runtime/API surfaces that depend on it, so the result is usable end to end rather than partially integrated.

**Why upstream-first for SDK.**  
OpenAPI code generation alone is not sufficient as a grant outcome and does not guarantee ecosystem convergence. Upstreaming credentials support into the existing wallet SDK path reduces duplication and improves adoption.

**Why this proposal fits Development Fund goals.**  
The output is reusable, open, and ecosystem-wide. It strengthens shared infrastructure and reduces duplicated implementation work across wallets, explorers, and dApps.

**Why PixelPlex.**  
PixelPlex operates CCView and Console Wallet — two of the primary consumers of this standard — and is already engaged in the review of PR #204. PixelPlex is also the author of CIP #169 (Party Profile Credentials), which builds on top of the Canton Network Credentials Standard, so the team has an ongoing interest in keeping this integration layer healthy beyond the grant period.