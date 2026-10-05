## Development Fund Proposal

**Author:** PixelPlex Inc. (Nikita Gerasimenok, Vladislav Demidovich)
**Status:** Draft
**Created:** 2026-10-05
**Label:** token-asset-standards

**[Champion](https://github.com/canton-foundation/canton-dev-fund/blob/main/sig-directory.md):** Need Champion

---

## Abstract

Today an outside backend that wants to integrate a Canton token learns three things by personal message from the token author: which packages the token needs, when a breaking change is coming, and which published token those facts belong to. This proposal funds the **Token Publication Standard**: a CIP, a normative HTTP API, an on-ledger publication-registration Daml package, an open-source reference publisher server, and a client-side verifier, followed by a MainNet launch with real publishers and integrators.

The standard is additive and opt-in per token. It adds one identifier, the contract id of a small publication-registration contract, and leaves CIP-0056 holdings, transfer, allocation, and settlement unchanged. A client verifies the contract id on its own participant by exercising a no-op choice, so the result does not depend on trusting the publisher's server.

The normative documents already exist as drafts and are part of this submission:

- [cip-token-publication-standard.md](cip-token-publication-standard.md): the CIP (meaning, registration contract, verification).
- [token-publication-http-api.md](token-publication-http-api.md): the normative HTTP API for version 0 (paths, JSON shape, errors).

This grant funds taking those drafts to a ratified CIP and a working, audited, adopted ecosystem asset.

---

## Specification

### 1. Objective

Give every Canton token author a standard, trust-minimized way to publish what an integrating backend needs (packages, version, announced breaking changes, and a verifiable published-token id), and give every integrator one way to consume it, so that integrating a new token no longer requires bilateral coordination.

The single objective is **adoption of one publication standard**. All deliverables below serve it: the specification, the on-ledger package, the reference server, the client verifier, and the launch with real publishers and integrators.

### 2. Implementation Mechanics

**Publication facts.** For each published token the publisher exposes: the package ids a client must obtain and the DARs containing them (marked `current`, `planned`, or `deprecated`); a version label with a note of what changed; migrations (announced changes an integrator has to act on, including changes that are not a new package, such as a new required key in a free-form argument map); and the publication-registration contract with its defining package. A client checks package ids against the published set before uploading to its own node and decides itself whether to vet.

**Publication registration.** The publisher creates one `PublicationRegistration` contract per published token, signed by the registrar, with `instrumentId` and `instrumentAdmin`. Its contract id is the published token's id inside this standard. It has one non-consuming, effect-free choice, `PublicationRegistration_Ping`. No token flow depends on this contract.

**Independent verification.** A client takes the disclosed contract and its defining package, audits the (deliberately tiny) package, uploads it to its own participant, and exercises the no-op choice. Canton checks vetting, the signatory, and the signature over the original create. Success means the contract was created under the registrar's authority without going back through the issuing server.

**Restricted reads.** Holdings, activities, holders, and updates routes must exist for every published token. The publisher chooses per read to serve it, not serve it (`403 access_not_granted`), or serve it with a token and say where to get one (`403` with `tokenUrl`). Packages, version, migrations, package status, and the registration are always served without a token.

**Work to be delivered by this grant:**

| Component | Description | Technology |
| --- | --- | --- |
| Specification | Finalize the CIP and HTTP API v0 through the CIP process; incorporate reviewer feedback; obtain a CIP number | Markdown, OpenAPI 3.x description of the HTTP API |
| Registration package | `PublicationRegistration` Daml template and `PublicationRegistration_Ping` choice; tests; DAR published with its package id | Daml, Daml Script |
| Reference publisher server | Open-source server implementing the full HTTP API v0: catalog, token, version, migrations, package status, packages, validate, proof, restricted reads with the three access choices; works for a utility registry token and a CIP-0056 token | TypeScript (Node.js); Ledger/JSON API; pluggable storage |
| Conformance suite | Black-box test suite that any second publisher implementation can run to show it satisfies the CIP and HTTP API | TypeScript, runnable in CI |
| Client verifier | Library and CLI that perform the full client flow: read facts, validate package ids, audit-and-upload helper, exercise the no-op choice on the client's own participant | TypeScript; usable from any backend |
| Security review | Independent review of the registration package and the reference server (restricted-read handling, DAR proxying, input validation) | External auditor |
| Documentation and onboarding | Publisher guide, integrator guide, migration-announcement guide, worked example against LocalNet and DevNet | Markdown |

**Operational approach.** The reference server is a stateless HTTP service over a publisher's own ledger node and a small data store. It does not prepare, execute, or submit end-user transactions. Hosting layout is explicitly outside the standard, so other implementations stay interoperable by satisfying the conformance suite.

**Effort estimate (basis for the funding request).**

| Milestone | Estimated effort | Notes |
| --- | --- | --- |
| M1 | ~4 person-months | Spec finalization, CIP process, Daml package and tests |
| M2 | ~9 person-months | Server, conformance suite, client verifier, LocalNet and DevNet flows |
| M3 | ~4 person-months + external audit | Audit remediation, MainNet launch, docs, onboarding of first publishers |
| M4 | ~5 person-months | Integrator and publisher onboarding support, migration tooling, 12 months of maintenance commitment set up |
| **Total** | **~22 person-months** + audit | Funding below is approximately 38,000 to 39,000 CC per person-month, audit included |

### 3. Architectural Alignment

- **CIP-0056 (Token Standard):** The CIP `Requires: 0056`. A CIP-0056 instrument remains the pair `(instrumentAdmin, instrumentId)`. Holdings, transfer, allocation, and settlement do not store the publication contract id and do not depend on it. The standard sits beside CIP-0056, in the area it does not cover: how an outside backend learns which packages a token needs and what is changing.
- **Utility registry surfaces:** The same publication id names a published token on a utility registry surface and on a CIP-0056 surface, giving integrators one identifier across both.
- **Canton package vetting and authorization model:** Verification relies entirely on Canton's own checks (package vetting, signatory authorization) on the client's participant. No new trust root is introduced.
- **Ecosystem priority:** Reducing integration cost for new tokens directly supports more applications supporting more tokens, and keeps tokens usable in flows they were not built for.

### 4. Backward Compatibility

*No backward compatibility impact.* The standard is additive and opt-in per token. Existing instrument identifiers are unchanged. Clients that never read publication facts are unaffected. A publisher that does not advertise the `supportedApis` flag is not required to expose anything, and clients that do not recognize the flag ignore it. Changing the meaning of the registration check, or dropping a listed publication fact, would be a breaking change and would be a new version of both documents.

---

## Milestones and Deliverables

### Milestone 1: Ratified Specification and Registration Package
- **Estimated Delivery:** Month 2 from approval
- **Focus:** Turn the existing drafts into a reviewed CIP and publish the on-ledger registration package.
- **Deliverables / Value Metrics:**
  - CIP submitted to the CIP process, reviewed publicly, with a CIP number assigned and status at least Proposed. Open reviewer comments are resolved or answered in the PR.
  - HTTP API v0 published as an OpenAPI description consistent with the CIP.
  - `PublicationRegistration` Daml package with tests, DAR published, package id recorded in the CIP.
  - Written feedback from at least 2 external parties (a token publisher and an integrating backend or wallet) on the draft, with the resulting changes recorded.

### Milestone 2: Reference Publisher, Conformance Suite, and Client Verifier
- **Estimated Delivery:** Month 5 from approval
- **Focus:** Working open-source implementations of both sides of the standard.
- **Deliverables / Value Metrics:**
  - Open-source reference publisher server implementing every route in HTTP API v0, serving both a utility registry token and a CIP-0056 token.
  - Conformance suite that passes against the reference server and runs in CI.
  - Client verifier library and CLI that perform the full flow (facts, package validation, no-op choice exercise on the client's own participant).
  - End-to-end demonstration on LocalNet and DevNet, including a migration announcement that is detected and acted on by the client.
  - Repositories under an open-source license (Apache-2.0 proposed) with public issue trackers.

### Milestone 3: Audit and MainNet Launch with First Publishers
- **Estimated Delivery:** Month 6 from approval
- **Focus:** Security review, production readiness, and first real publications.
- **Deliverables / Value Metrics:**
  - Independent security review of the registration package and reference server, with findings remediated and the report published.
  - Publisher and integrator guides published.
  - At least 2 tokens (including a token operated by an organization other than PixelPlex) published on MainNet under the standard, each with a registration contract that a third party has verified using the client verifier.

### Milestone 4: Ecosystem Adoption
- **Estimated Delivery:** Month 9 from approval
- **Focus:** Demonstrated use by independent publishers and integrators.
- **Deliverables / Value Metrics:**
  - At least 5 distinct tokens published on MainNet by at least 3 distinct publishers.
  - At least 3 independent integrators (wallets, custody backends, or exchanges) consuming publication facts or running the registration check in production or a documented pilot.
  - At least 1 second-party or third-party publisher implementation, or a documented adopter running their own deployment, passing the conformance suite.
  - Public maintenance commitment in place (see Acceptance Criteria).

---

## Acceptance Criteria

The Tech & Ops Committee will evaluate completion based on:

- Deliverables completed as specified for each milestone
- Demonstrated functionality or operational readiness
- Documentation and knowledge transfer provided
- Alignment with stated value metrics

Project-specific acceptance conditions:

- **M1** is accepted when the CIP has a number, has been publicly reviewed, and the external feedback from at least 2 parties is recorded.
- **M2** is accepted when a Committee reviewer, with no PixelPlex involvement, can run the conformance suite and the client verifier against the reference server on DevNet and reproduce the demonstrated flow.
- **M3** is accepted on the basis of live MainNet publications that a third party has verified, not on the existence of a report or a deployment.
- **M4** is accepted on the basis of named, independently confirmed adopters (publishers and integrators). Adopters may be listed by name, or confirmed to the Committee privately where they prefer not to be named publicly.
- **Sustainability:** PixelPlex commits to maintaining the reference server, registration package, conformance suite, and client verifier for 12 months after M4 acceptance (bug fixes, dependency and Canton/Splice version updates, issue triage). The code and standard are public goods, are not tied to a PixelPlex-hosted service, and may be maintained by any party after that period.

---

## Funding

**Total Funding Request:** 850,000 CC

### Payment Breakdown by Milestone
- Milestone 1 _(Ratified Specification and Registration Package)_: 120,000 CC upon committee acceptance
- Milestone 2 _(Reference Publisher, Conformance Suite, and Client Verifier)_: 330,000 CC upon committee acceptance
- Milestone 3 _(Audit and MainNet Launch with First Publishers)_: 150,000 CC upon committee acceptance (includes the external audit cost)
- Milestone 4 _(Ecosystem Adoption)_: 250,000 CC upon final release and acceptance

### Volatility Stipulation
The project duration is **greater than 6 months** (about 9 months). The grant is denominated in fixed Canton Coin and will require a re-evaluation at the 6-month mark.

---

## Co-Marketing
Upon release, the implementing entity will collaborate with the Foundation on:

- Announcement coordination
- Case study or technical blog
- Developer or ecosystem promotion

Specific commitments:

- A technical blog post at the M2 release explaining the standard and the no-op-choice verification, written for integrator engineers.
- A publisher onboarding walkthrough (written and recorded) at M3.
- A joint announcement and a short adopter case study at M4, naming adopters who agree to be named.
- A presentation of the standard at one Canton ecosystem developer call or meetup.

---

## Motivation

A Canton token is not usable from outside its author's node unless a backend has the packages, knows how the token behaves, and can tell it apart from another token with the same name. Today the author's developers and the backend's developers arrange all three by personal message. That fails in the same way for three groups: authors have no standard way to expose the packages their contracts need; backends must learn each token from scratch and again at every change; and users inherit the cost, since applications support few tokens and tokens get used in flows they were not built for.

**Ecosystem impact.** The standard applies to every token that wants to be integrated by parties other than its author. Wallets, custody backends, exchanges, and application backends that integrate more than one token all benefit, because the publication surface is the same for each token. We expect the benefit to concentrate in two groups: token publishers who want broader integration, and integrators who handle several tokens. As an order-of-magnitude estimate, we expect a meaningful share of tokens that integrators handle beyond one or two well-known tokens to adopt it within the first year, which is why M4 sets concrete targets (at least 5 tokens, 3 publishers, 3 integrators) rather than a percentage of the ecosystem.

**Public good.** The CIP and HTTP API are CC0. The reference server, conformance suite, registration package, and client verifier are open source. Any publisher can run its own deployment or write its own implementation and prove interoperability with the conformance suite.

**Sustainability.** See the maintenance commitment in Acceptance Criteria.

---

## Rationale

**Why a new standard rather than extending CIP-0056.** CIP-0056 identifies an instrument by `(instrument admin, instrument id)` and defines how to hold, transfer, allocate, and settle it. It does not describe which packages a published token needs, or which change is coming, and it does not provide one identifier that means the same thing for a utility registry token and a CIP-0056 token. Adding a publication field to holdings or transfer would change flows that work today and would affect every implementation. This standard leaves CIP-0056 untouched and builds beside it. It is an extension of the existing ecosystem rather than a replacement, and it explicitly requires CIP-0056.

**Why an on-ledger registration contract with a no-op choice.** A contract id returned over HTTP is only the server's claim. Exercising a non-consuming, effect-free choice makes the client's own participant check vetting and the signatory's authorization, so verification does not depend on the issuing server. The package stays tiny so that auditing it takes minutes.

**Why version, package status, and migrations are separate.** Package status says what package ids are `current`, `planned`, or `deprecated`. It does not say what an integrator has to change, and a new required key in a free-form argument map is not a new package id at all. The version label names current behavior; a migration says an integrator has to act, by when, and how. Each leaves a gap without the other two.

**Why restricted reads are optional.** Packages, version, and the registration are enough to integrate a token. Holder and history reads concern holders, and a publisher can keep them off the public internet and still be usable. The standard requires the routes to exist and the publisher to answer them honestly (`403 access_not_granted`) instead of hiding them, so clients can tell "not served" from "unknown" or "empty".

**Why not a directory of base URLs.** Listing every base URL on the network is a different problem, comparable to a public list of RPC endpoints. The standard leaves it to community lists.

**Alternatives considered.**
- An off-ledger signed list of token ids was rejected: it moves the trust root off the ledger, which the registration check exists to avoid.
- Standardizing every free-form argument map so migrations are unnecessary was rejected: tokens differ in those maps on purpose. Migration announcements tell clients about a change without forbidding it.
- Standardizing server hosting, language, or layout was rejected: it is a product question, not an interoperability question. Two publishers interoperate when they expose the same facts and the same registration check, which the conformance suite tests.
