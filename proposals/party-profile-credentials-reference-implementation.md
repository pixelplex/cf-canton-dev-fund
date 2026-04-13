## Development Fund Proposal

**Author:** PixelPlex Inc. (info@pixelplex.io)
**Status:** Draft  
**Created:** 2026-04-13  
**Label:**
- wallet-apps

---

## Abstract

This proposal funds an **open-source reference implementation** of the **Canton Network Party Profile Credentials** standard (draft in the CIPs repo, e.g. [PR #169](https://github.com/canton-foundation/cips/pull/169): profile metadata as CN Credential claims — display name, avatar, website, contact fields - plus application-side rules to derive a single effective profile from credentials issued across registries and issuers).

Deliverables are intended as **ecosystem public goods**: a reusable library (and supporting examples/docs) so wallets, explorers, and dApps can **publish, update, and render** consistent party profile claims without each team reimplementing parsing, construction, namespacing, and merge semantics. Work stays aligned with the **Canton Network Credentials** model. The CIP defines the normative claim keys and application-side behavior for this scope.

---

## Specification

### 1. Objective

**Problem:** Party IDs are correct for infrastructure but poor for human UX. Teams need a **shared, predictable way** to store and resolve **profile UI data** (names, avatars, links) on top of credentials; today that behavior is ad hoc.

**Outcome:** Ship a **maintained reference implementation** (library + tests + docs + minimal integration examples) that encodes the CIP’s claim keys and resolution rules, so any application can adopt the standard with minimal integration cost.

### 2. Implementation Mechanics

**Components (planned):**

1. **Profile claims module (core + contact/social)** — Parse and validate **all** profile-related claims under the CIP namespace (e.g. `cip-<nr>/displayName`, `cip-<nr>/avatar`, contact/social keys as listed in the CIP) and map them to a stable in-memory model for UI consumption. Naming of keys tracks the published CIP text as it evolves (including any later alignment of social key names in the CIP itself).
2. **Claim construction (write path)** — Helpers to **set / publish** profile data: build a valid claim map from user or app input (strings, URLs, handles), apply the same validation rules as on read, and expose a structure the host can attach when **issuing or updating** a CN Credential. **Signing, ledger submission, and which registries allow self-issuance (`issuer = holder`) or third-party issuers** follow the **Canton Network Credentials** model and the deployment’s registry policy — this library only ensures claim payloads and normalization (e.g. trimming, format checks) match the Party Profile CIP.
3. **Resolution helper** — Implement the CIP’s **application-side** merge for the **read** path: ordered sources `(registry, issuer)` (including `self`), last-write-wins within a source where applicable, key-by-key merge across sources — with extension points so host apps can plug in their own credential-fetch and registry logic.
4. **Testing & fixtures** — Unit tests and credential fixtures representing realistic multi-issuer scenarios (including round-trip: build -> validate -> merge).
5. **Examples** — Small sample (e.g. example app or CLI snippet) showing **both** preparing claims for publication and consuming merged profiles from CN Credential–shaped data.
6. **Documentation** — Developer-facing docs: API overview, claim catalog, read vs write flows, merge semantics, and integration notes for typical wallet/dApp flows.

**Technology:** Language and packaging to be chosen for broad adoption (e.g. TypeScript or Rust/Go — **to be fixed in milestone planning** with the champion/committee); artifacts published under an **Apache-2.0** (or committee-preferred OSI) license in a public repository.

### 3. Architectural Alignment

- Builds on the **CN Credentials** abstraction (claims as key–value metadata tied to holder/issuer semantics) — no protocol change to Canton core.
- Implements the **Party Profile Credentials CIP** for its scope: **constructing** profile claims (write path), **consuming and merging** them (read path), and the same validation rules on both sides. Other ecosystem standards may address broader naming or resolution; this library does not need to embed them to be useful.
- Supports ecosystem priorities under **App Building and Developer Experience** (interoperable wallets/dApps, shared standards) and **party portability** (consistent human-facing party representation).

### 4. Backward Compatibility

Additive only: applications that do not adopt the library continue as today. Claim keys follow the CIP; if the CIP number or key names change before finalization, the implementation will be updated in lockstep — no breaking change to Canton protocol or ledger contracts by this grant.

---

## Milestones and Deliverables

### Milestone 1: Foundation — library skeleton, claim model, read + write helpers, tests

- **Estimated Delivery:** Month 2 (from kickoff)  
- **Focus:** Public repo, CI, core data model, claim **parsing and construction** / validation for the profile namespace, initial test suite.  
- **Deliverables / Value Metrics:** Published OSS repo; passing CI; documented claim catalog matching the CIP draft; ≥80% line coverage on core parsing/validation/build (target).

### Milestone 2: Resolution + end-to-end examples

- **Estimated Delivery:** Month 4  
- **Focus:** Merge/resolution helper per CIP; end-to-end example integration (build claims -> hand off to host issuance path -> fetch -> merge); expanded fixtures covering multi-source scenarios from the CIP examples.  
- **Deliverables / Value Metrics:** Working example flow; README walkthrough; at least one external pilot integrator **or** documented integration steps for one reference wallet pattern (committee can help identify a candidate).

### Milestone 3: Docs, hardening, handover

- **Estimated Delivery:** Month 5  
- **Focus:** Performance/security pass on non-cryptographic edge cases (oversized strings, invalid URIs), documentation polish, maintenance plan.  
- **Deliverables / Value Metrics:** Final docs; release version tag; short “operations & maintenance” section (issue handling, release cadence for the grant period).

---

## Acceptance Criteria

The Tech & Ops Committee will evaluate completion based on:

- Deliverables completed as specified for each milestone  
- Demonstrated functionality or operational readiness  
- Documentation and knowledge transfer provided  
- Alignment with stated value metrics  

**Project-specific:**

- Library implements the published CIP claim keys and merge semantics (as merged to the CIPs repo or as agreed with reviewers during the grant).  
- **Write path:** building and validating claim payloads for publication matches the CIP; **read path:** merge and display behavior matches the CIP.  
- Contact/social fields behave according to the CIP’s normative keys and rules.  
- Open repository, license, and tagged release(s) suitable for ecosystem reuse.

---

## Funding

**Total Funding Request:** _TBD — to be agreed with Tech & Ops (CC amount and schedule)._  

### Payment Breakdown by Milestone

- Milestone 1 — Foundation: _XX_ CC upon committee acceptance  
- Milestone 2 — Resolution + end-to-end examples: _XX_ CC upon committee acceptance  
- Milestone 3 — Docs + hardening: _XX_ CC upon final release and acceptance  

### Volatility Stipulation

If the project duration is **greater than 6 months**:  
The grant is denominated in fixed Canton Coin and will require a re-evaluation at the 6-month mark.

If the project duration is **under 6 months**:  
Should the project timeline extend beyond 6 months due to Committee-requested scope changes, any remaining milestones must be renegotiated to account for significant USD/CC price volatility.

---

## Co-Marketing

Upon release, PixelPlex will collaborate with the Foundation on:

- Announcement coordination  
- Case study or technical blog post describing the standard and the reference library  
- Developer-facing promotion (e.g. ecosystem channels, workshop if useful)  

---

## Motivation

Consistent **party profiles** reduce phishing and confusion, improve trust in UIs, and lower integration cost for every wallet and explorer. A **reference implementation** turns a CIP from paper into **reusable code**, accelerating adoption and reducing fragmentation — a clear **common good** for Canton.

---

## Rationale

**Why a dedicated implementation grant:** Standards without code lag; wallets need copy-pasteable, tested building blocks.

**Why this CIP and this grant:** The Party Profile CIP defines **what** profile fields mean and how to merge **profile claims** in credentials for display. This grant funds a **direct implementation of that CIP** so teams can ship UX without waiting for unrelated larger efforts.

**Alternatives considered:** Each dApp reimplements parsing (high error rate); or only proprietary implementations (weak ecosystem interoperability). Open reference code under a permissive license is the most scalable path.
