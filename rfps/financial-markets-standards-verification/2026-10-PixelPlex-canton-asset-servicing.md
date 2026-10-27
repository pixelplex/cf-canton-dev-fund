## Development Fund Proposal

**Organization:** PixelPlex
**Author / Primary Contact:** Semyon Protas, Yury Korzun (PixelPlex)
**Status:** Draft
**Date:** 2026-10-27
**Proposal Type:** RFP-aligned
**RFP / Roadmap Area:** Financial Markets, Standards & Verification. RFP 12 RWA Standards, sub-area "Daml and Institutional RWA Workflow Standards": item 4 (corporate actions and other asset lifecycle events), item 7 (repo, collateral, lending and servicing workflows), item 8 (interoperability between Canton applications), item 11 (conformance tests and reference implementations).
**Champion:** `Needs Champion`
**Total Funding Request:** 2,400,000 CC
**Project Duration:** 28 weeks
**Maximum Amount:** `N/A`
**Maximum Duration:** `N/A`
**Label:** token-asset-standards, rfp-12:rwa-standards
**Suggested SIGs:** Token Standards / Asset Standards (primary), Financial Workflows & Composability, Regulatory Compliance

---

## Abstract

We propose **Canton Asset Servicing**, an open standard and reference implementation for what happens to a tokenized security after it is issued and transferred: coupon and dividend payments, redemption, and corrections when an announcement was wrong, including for a security that is pledged.

The Canton Network Token Standard covers who holds what and how it moves. CIP-0112 added accounts, custody relationships and a holdings event log, which a securities registry needs. Neither CIP-0056 nor CIP-0112 defines servicing on top of that. There is no standard way to announce that an instrument pays on a date, to state who is entitled to the payment as of a record time, to record what has been paid against it, to redirect it when the asset is pledged, or to correct an event after money has moved. The terms "record date" and "entitlement" do not appear in either CIP.

Each registry therefore defines its own templates, and a custodian, collateral application or wallet from another vendor cannot read them. For a pledged bond, the recorded holder and the party owed the coupon are different, and the standard has no way to say so. Institutions care most about this case.

This proposal delivers a CIP and a Daml interface package, `splice-api-token-servicing-v1`. That name and the name of the conformance package are working names: `splice-*` is the Splice namespace, and the final names are for the Token Standards SIG to decide. The package adds three objects to the token standard as five interfaces: **lifecycle event**, **entitlement** and **entitlement redirect**. Alongside it we deliver the off-ledger registry API that the standard's conventions require, a reference bond registry, a conformance test package in the form of the existing V1/V2 test harnesses, and a demonstration. In the demonstration a registry and a custodian, each on its own participant, process a coupon on a pledged bond, including a corrected event, and produce matching entitlement records. 300,000 CC of that milestone is paid only if at least one of the two applications is neither operated nor authored by PixelPlex.

The proposal introduces no new payment rail. Money moves over the existing `TransferInstruction` and `Allocation` flows, and redemption over the `cip-112/burn` account. What the standard adds is that a payment and its record happen in one transaction, so an entitlement cannot be paid twice or marked paid without being paid. Everything is Apache-2.0.

A companion implementation draft is filed alongside this proposal as [`2026-10-PixelPlex-canton-asset-servicing-implementation-draft.md`](./2026-10-PixelPlex-canton-asset-servicing-implementation-draft.md): the interfaces and their Daml code, the derivation and settlement rules, the privacy analysis, the demonstration setup and the conformance scenarios.

---

## Specification

### 1. Objective

Make servicing of a tokenized security work across applications from different vendors, as transfers already do.

A registry that issues a bond on the token standard can move it. To service it, the registry today defines its own contracts for the coupon announcement, its own rule for who gets paid, its own record of payments, and its own way of passing all three to the custodian and the collateral application on the other side of the custody relationship. These definitions are private to the registry, so counterparties reconcile by file, by API or by hand.

We found five gaps, each checked against the standard as shipped in Splice 0.8.3:

1. **No event object.** There is nowhere to state "instrument X pays 2.5 per 100 units, record time 2026-12-29T22:00Z, payment time 2026-12-31". Each registry models this privately.
2. **No entitlement.** Payment is owed to holders as of a record time. Holdings are per-party contracts and the registry sees them, but nothing expresses the derived result (who is owed how much, how it was computed, and how much has been paid) as an object that the payer, payee, custodian and auditor all read the same way.
3. **Locked holdings carry no economic meaning.** `Lock` in `HoldingV2.daml:48` has four fields: `holders`, two expiry fields (`expiresAt`, `expiresAfter`), and a free-text `context` meant for wallet display. A pledged holding cannot say which obligation it secures, or that the coupon belongs to someone other than the recorded holder.
4. **No correction path.** When an announcement was wrong and payments have been made, there is no standard link between the wrong event, the corrected event, what was paid, and what is still owed.
5. **No shared arithmetic.** Each party computes rounding and withholding on its own, so two correct implementations can produce different numbers and fail to reconcile on the last decimal.

**Version 1 covers one instrument's life:** cash distributions, full redemption, a pledge redirect, payment with a record of what has been paid, and corrections. Holder elections, unit changes, partial redemption, residual allocation against a funded total, securities-lending equivalent payments, and mergers, spin-offs or rights issues are **extensions**: separate packages on top of this one, proposed separately. An extension reuses version 1's keys, leaves its interfaces as they are, and carries what it adds in interfaces of its own.

Left to adjacent work: valuation and risk parameters (Canton Risk Engine #665), liquidation mechanics (Canton Common Asset #232), incentive and rewards distribution (#806), sub-allocation inside a custodian's omnibus account, and the legal enforceability of any agreement a redirect refers to. This proposal standardizes the servicing record, its derivation and its settlement. It does not define the financial products around them.

### 2. Implementation Mechanics

**Where it attaches.** CIP-0112 (approved 2026-06-12) provides the substrate, and we build on it. `Account { owner, provider, id }` makes the custody relationship explicit, and the special accounts `cip-112/mint` and `cip-112/burn` let issuance and redemption use the ordinary transfer and allocation flows. Our interfaces follow the V2 conventions: a factory contract for the capability a non-admin initiates, a `_PublicFetch` choice taking `actors : [Party]` so an app can safely read a contract obtained from an untrusted source, and `ExtraArgs` carrying a registry-provided `ChoiceContext` fetched over an off-ledger API.

**The three objects.**

- *Lifecycle event* (`LifecycleEvent`). Only the instrument admin may announce an event, so it needs no factory. It carries the instrument, an event id, the event kind (cash distribution or redemption), the announcement time, the **record time**, the **payment time**, the rate per unit and the instrument the payment is made in, the rounding rule, the synchronizer its record time refers to, a status and, for a correction, a reference to the event it supersedes. Its observers are kept narrow. A contract's stakeholders can see each other, so an event observed by every holder would disclose the holder register. The event reaches its audience in two other ways: as plain data from the registry's announcement endpoint before the record time, for display, and as terms copied onto each holder's own entitlement after derivation. The copy on the entitlement is authoritative.

- *Entitlement*. The derived result, one per event, holding account and slice. A slice is either the holder's own share or the share allocated under one redirect. The entitlement records the units attributed to it, the gross amount, each deduction with the basis it was applied to, the net amount, the account it is payable to, the terms it was derived under, and **how much of it has been paid**. Its observers are the account parties of both accounts, as the standard's `accountParties` helper defines them, plus the paying agent, so a custodian sees its clients' entitlements the same way it sees their holdings. It has one state-changing choice, `Entitlement_Settle`, which makes the payment and records it in the same transaction. That is what prevents a retry after a crash from paying twice.

- *Entitlement redirect*. This is what makes pledged assets work. A redirect is scoped to an account and an instrument, with an optional cap on units. It is not tied to a holding contract, because holdings are archived and re-created on every transfer, split and lock. It states that income derived from what the account holds at an event's record time, for a given class of events and over a given period, is owed to a different account, the pledgee's. It can reference the pledge agreement in whichever application holds it, by contract id only. A redirect is built the way a V2 transfer instruction is. Any one of the holder's account parties proposes it. Every further approval, from the pledgee's side and from any other holder-side party the registry's account policy requires, is a separate single-party command on a pending instruction, in any order. The holder, its custodian, the pledgee and the pledgee's custodian can therefore each act from their own participant. The principals (owners) of both accounts always approve, so a custodian cannot redirect a client's income on its own. The registry's authority comes from the factory the proposal is exercised on, so the registry does not need a separate step. A pledgee-side party can revoke alone; a holder-side request to revoke waits for the pledgee's side. The collateral application asks the holder's side to propose, approves from the pledgee's side, and needs no knowledge of the registry beyond the standard.

**How payment happens.** The paying agent exercises `Entitlement_Settle`, and inside that choice the registry moves the amount. A cash distribution is a `TransferFactory_Transfer` from the paying agent to the payable account. A redemption is paid the same way and in full. From shortly before its record time (by the ledger-time tolerance) the instrument is frozen, so the units can only go to the `cip-112/burn` account, where the holder's own allocation retires them. If that allocation already exists, the burn happens in the same transaction as the cash payment: the delivery-versus-burn flow that CIP-0112 describes and the shipped V2 test performs. The same transaction re-creates the entitlement with its paid amount increased. Payment and record therefore cannot diverge, partial payments of a distribution add up, and the ledger orders a correction that races a payment. Every movement also carries the entitlement's opaque reference in its `meta`, so the receiving side can reconcile without a separate file.

**Deterministic arithmetic that each party can check.** The standard fixes the computation as well as the data shape: the record-time snapshot of holdings and of redirects, the rounding, the permitted scales and directions, and the deduction bases. The rounding residual stays with the registry. Each holder's figure is then a function of that holder's units alone, so a custodian that sees only its own clients' positions can recompute every line for them. §5 of the implementation draft gives the rules, compiled against the Daml-LF target the token standard uses. Test vectors are published at Milestone 2, before any implementation is final.

**Corrections.** Nothing is erased or changed in place. A correction to the terms is a new event that supersedes the old one. A correction to the record time, the event kind or the payment instrument is instead a cancellation followed by a new announcement, because each of those changes who is entitled, which slices exist, or what an earlier payment was made in. Affected entitlements are superseded too: the superseded record is re-created with the key of its replacement, and the replacement carries the key of the record it supersedes, so a counterparty reading its own participant can follow the link both ways. The replacement inherits what has already been paid, and what is still owed is the difference: 100 owed, 40 paid, corrected to 90, leaves 50. The compensating payment is an ordinary settlement. An overpayment shows as a negative outstanding amount. It is recorded as a claim and is never paid as a negative transfer, since nothing on the ledger can force funds to be returned.

**Off-ledger API.** `servicing-v1.yaml` follows the shipped registry-API shape under `/registry/servicing/v1/…`, with reads, factories and choice contexts. It lists announced and historical events for an instrument as plain data, returns the redirect factory with its choice context, and serves choice contexts for acting on a pending redirect, revoking a live one, and settling an entitlement through a delegated paying agent. It has no endpoint that lists an account's entitlements. Holdings and entitlements are both read from the Ledger API, and a second source for the same figures would bring back the reconciliation problem this proposal is meant to remove.

**Deliverables.** The CIP; the Daml package `splice-api-token-servicing-v1` with its OpenAPI; a reference bond registry on the same test-harness pattern as `splice-token-standard-v1-test`; a conformance package, `splice-token-servicing-test`, which a third-party registry copies and runs against its own implementation; a mapping and an adapter from Daml Finance lifecycle effects; and a TypeScript library with CLI commands for wallets and operators to list events and entitlements and report reconciliation differences.

### 3. Architectural Alignment

- Additive to CIP-0056 and CIP-0112. No existing interface changes, no existing package gets a new major version, and a registry that does not implement servicing is unaffected.
- Uses the standard's own extension points: factories, `_PublicFetch` in its V2 form, `ExtraArgs` / `ChoiceContext`, and `Metadata` keys prefixed with the defining app's DNS name, as `MetadataV1` requires.
- Follows Canton's privacy model. The registry already sees every holding of its instrument, so deriving entitlements discloses nothing new to it. A redirect tells the registry that a pledgee exists and who it is, which the registry needs in order to pay it. The terms of the pledge stay in the application that owns them and are referenced by contract id only. The pledgee sees the entitlements redirected to it. When it approves or revokes, it also learns how much of the position is already committed to other redirects, as an aggregate with no identities. It does not see the holder's position, with two exceptions that the implementation draft lists among the disclosures put to the Milestone 1 gate: under an uncapped redirect its entitlement equals the position, and a pledgee whose slice falls short of its cap can work out the position.
- Supports multiple issuers and applications, as RFP 12 requires. Nothing in the design assumes a single registry, a single custodian or a service run by PixelPlex.
- Works with Daml Finance. A registry that models its bond in Daml Finance keeps lifecycling it there, and an adapter turns the resulting effects into servicing events and entitlements. The implementation draft (§3.1) gives the mapping, and Milestone 1 publishes it field by field against the current release.

### 3a. Relationship to adjacent work

Our scope is **the cross-application servicing record, its derivation and its settlement**: the event, the entitlement and what has been paid on it, and the redirect that follows a pledge.

| Adjacent work | Overlap | Boundary |
| --- | --- | --- |
| Token Standard V2 / CIP-0112 (Digital Asset, approved and funded) | accounts, custody relationships, event log, mint/burn accounts | we build on top of it, propose no competing version of any V2 interface, and take the CIP through the same Token Standards SIG |
| Token Data Disclosure Standard #878 (PixelPlex, open) | a default token supporting CIP-0056 and CIP-0112, and a token test kit | #878 covers how a token is published and installed, with a default implementation; this proposal covers how any token is serviced. The Milestone 3 reference bond registry uses #878's default token as its holding layer if it has shipped, and otherwise builds the minimal holding layer the suite needs. The servicing interfaces, register, derivation, settlement and suite are funded only here |
| Canton Common Asset #232 (open) | lock, unlock, liquidate primitives | we use locks and do not define liquidation. A redirect is one way to decide who gets the coupon while a holding is locked; whether #232 needs it is for that team to decide |
| Canton Risk Engine #665 (open) | risk parameters, valuation | valuation is out of scope here; entitlement records can be an input to their engine |
| alphaPilot #758, Vault Infrastructure #796, Merkl incentive distribution #806 (open) | asset discovery, redemption queues, programmatic distributions | these are applications that can consume events and entitlements; none defines them. An incentive payout measured on activity is derived differently from a claim read off a holder register at a record time |
| Tokenized Vault Standard #863 (Mystic, approved) | vault interfaces on CIP-56 and CIP-112, with deposit and redeem of vault shares | redeeming a vault share is an operation of the vault on its own shares, not a lifecycle event of a held security. A vault that holds bonds is, for this standard, an ordinary holder, and its entitlements are paid to it like any other |
| Daml Finance (Digital Asset) | lifecycle rules and effects for bonds and equities | a library used inside one registry's application; we define what a second application reads. We map to it and do not replace it |
| `ex-asset-servicing` (Digital Asset, archived, SDK 1.18) | coupons, redemptions, dividends and splits, processed by each holder on its own position against per-unit effects published by the issuer | the precedent for these workflows, with the opposite design. Holder-side processing hides positions from the issuer, but under the token standard the holder cannot pay itself, the registry already sees every holding, and a pledgee would depend on the holder forwarding the coupon. It runs inside one application, cannot be read by a second one, and is not a current alternative. The implementation draft (§3.1) compares the two |

None of these defines an event or entitlement that a second application can read.

### 4. Backward Compatibility

Existing deployments are not affected. The package adds interfaces and does not modify `splice-api-token-holding-v2`, `-transfer-instruction-v2`, `-allocation-v2` or any other shipped package. A registry that adopts servicing does not force its wallets or counterparties to upgrade. An application that has not implemented the interfaces behaves as it does today and does not display entitlements it cannot read. Extensions are separate packages on version 1's keys, so adopting version 1 does not commit a registry to any of them. Interfaces evolve under the standard's own versioning practice, and the conformance suite pins behaviour per Splice release in CI.

---

## Milestones and Deliverables

### Milestone 1: Standard draft and feasibility gate
- **Estimated Delivery:** week 6
- **Focus:** CIP draft, interface definitions, the derivation specification (snapshot of holdings and redirects, order of operations, rounding, deduction bases, withholding as a declared input), settlement and correction semantics, and the Daml Finance mapping, field by field.
- **Deliverables / Value Metrics:** draft CIP submitted to and reviewed in the Token Standards / Asset Standards SIG. Written reviews from at least **3 organizations**, including at least one registry or issuer, one custodian or account provider, and one collateral application.
- **Feasibility gate, published either way:** entitlement derivation demonstrated over CIP-0112 accounts with providers, where the recorded holder, the account provider and the pledgee are three different parties. In addition, the five redirect disclosures listed in the implementation draft are put in writing to at least one custodian and at least one buy-side holder, and each is asked whether it would allow a client position to be pledged under them: yes or no, with reasons. The answers are published whatever they are.

  **If the gate fails**, the fallback design restricts the redirect's beneficiary in Milestones 2 and 3 to an account **owned by** the holder's own provider, so no pledgee is named on the ledger, and the Milestone 5 demonstration drops the cross-provider case. Milestone 5 is then reduced by 200,000 CC, which is not paid.

### Milestone 2: Daml package and registry API
- **Estimated Delivery:** week 12
- **Focus:** `splice-api-token-servicing-v1` (lifecycle event, entitlement with settlement, redirect) with `servicing-v1.yaml`, published under Apache-2.0.
- **Deliverables / Value Metrics:** the package builds against the current Splice release and the one before it. Published test vectors for the derivation cover rounding boundaries in every direction at scales 0 and 10, the overflow boundary, deductions on each basis, holders with zero entitlement, holdings and redirects on each side of the record time, partial and multiple redirects on one account, and a position below its committed caps. Any implementation can rerun the vectors from the repository.

### Milestone 3: Reference registry and conformance suite
- **Estimated Delivery:** week 18
- **Focus:** a reference bond registry that pays coupons and redeems at maturity, with pledge scenarios, plus the adapter that announces its events from Daml Finance lifecycle effects; and `splice-token-servicing-test` in the same form as the existing V1/V2 test packages. The reference registry needs a full token-standard V2 holding layer (holdings, transfer factory, allocation factory) before any servicing code runs. It uses #878's default token if that has shipped; otherwise the minimal layer is built and costed in this milestone.
- **Deliverables / Value Metrics:** the reference registry passes the full suite of twenty scenarios listed in §11 of the implementation draft. They cover at least: a coupon to a plain holder on a basic account; a coupon to a pledged holding with the pledgee at a different provider; holder, provider and pledgee as three parties, with the provider recomputing the split; partial and multiple redirects producing distinct entitlements; a redirect built one party per command; concurrent completing approvals, distinguishing contention from business rejection; revocation and expiry; the record-time snapshot of redirects; settlement in full, in parts, on retry after a committed payment, and concurrently; redemption of a pledged holding under a record-time freeze, to `cip-112/burn`; corrected events after partial and full payment, in both directions, and racing a payment; a cancellation; corrections of the record time, kind or payment instrument after payment, handled as cancellations; refusal to derive across synchronizers; and the rounding vectors. Each scenario asserts field-by-field equality on the fields that can be derived independently, not only that totals agree. The TypeScript library and CLI also ship in this milestone, listing events and entitlements and reporting reconciliation differences, so that the Milestone 4 wallet has a client to build on and the Milestone 5 comparison has the tool that performs it.

### Milestone 4: Independent implementation
- **Estimated Delivery:** week 24
- **Focus:** adoption by parties other than the author.
- **Deliverables / Value Metrics:** at least **one registry or issuer not operated by PixelPlex** implements the interfaces and passes the conformance suite, with published results. At least **one wallet** shows its users announced events and entitlements, including what has been paid on them. Both are confirmed in writing or by a public repository.

### Milestone 5: Cross-application demonstration and handover
- **Estimated Delivery:** week 28
- **Focus:** the end-to-end case this proposal is for, plus documentation and maintenance.
- **Deliverables / Value Metrics:** **a registry and a custodian application, each on its own participant, process a coupon on a pledged security, including a corrected event after partial payment, and produce matching entitlement records.** This is demonstrated on DevNet or TestNet with a published transcript that any reviewer can follow, in the setup fixed in §10 of the implementation draft: every account, its owner and provider, and what each party observes. If at least one of the two applications is neither operated nor authored by PixelPlex, the demonstration evidences the standard and earns the third-party portion of the milestone (see Funding). Also delivered: a getting-started guide, an integration guide for registries, a reference architecture, and a 12-month maintenance commitment from PixelPlex.

---

## Acceptance Criteria

The Tech & Ops Committee will evaluate completion based on:

- Deliverables completed as specified for each milestone
- Demonstrated functionality against local Canton, DevNet and an authorized MainNet participant where applicable
- Documentation and knowledge transfer provided
- Alignment with stated value metrics

Project-specific conditions:

- The CIP goes through the standard CIP process. If the Token Standards SIG directs the design differently, the milestones are adjusted with the Committee instead of being delivered against a superseded draft.
- "Matching entitlement records" means field-by-field equality of the figures the custodian derives *independently*: units at the record time, the split between the holder's share and each redirect, gross amount, deduction amounts recomputed from the declared rates on a gross or remaining basis, net amount, the payable account, and the amount paid on the holder's own share. The custodian computes them from its own participant and the announced event, and the published CLI compares them, each side running it on its own data. The custodian is the holder's account provider, which observes the holdings and every redirect on the position, so it can derive each of these fields. Deductions on a basis declared by the registry are reported, not derived. The pledgee checks the amount paid on a redirected slice against its own receipts. Comparing the registry's contract with itself does not count as a match.
- Adoption in Milestones 4 and 5 is evidenced by public repositories or written confirmation from the adopting organization. An adopter that is a PixelPlex customer counts only if it runs its own participant and implements independently, shown by commit history in its own repository with no PixelPlex-authored commits in the implementation, or by a signed statement naming the engineers.
- The third-party portion of Milestone 5 (300,000 CC; 100,000 CC under the gate fallback) requires at least one of the two applications in the demonstration to be neither operated nor authored by PixelPlex. The rest of Milestone 5 is accepted on the demonstration and handover either way (see Funding).
- All code, specifications, test vectors and documentation are public under Apache-2.0 throughout the project, not only at the end.
- If the design shows that a servicing case cannot be represented without changing an existing token standard interface, we publish that finding and raise it with the SIG instead of working around it privately.

---

## Funding

**Total Funding Request:** 2,400,000 CC

### Payment Breakdown by Milestone
- Milestone 1 (Standard draft and feasibility gate): 300,000 CC upon committee acceptance
- Milestone 2 (Daml package and registry API): 400,000 CC upon committee acceptance
- Milestone 3 (Reference registry and conformance suite): 650,000 CC upon committee acceptance
- Milestone 4 (Independent implementation): 450,000 CC upon committee acceptance
- Milestone 5 (Cross-application demonstration and handover): 600,000 CC upon final release and acceptance

### Effort basis

**81 engineer-weeks** over 28 weeks, roughly 18 person-months, or about 30,000 CC per engineer-week across the grant:

| Milestone | Engineer-weeks | Effort |
| --- | --- | --- |
| 1 | 13 | an architect throughout, a second Daml engineer, financial-domain review, the gate interviews and their write-up |
| 2 | 14 | the interface package, the OpenAPI, the test vectors, CI against two Splice releases |
| 3 | 30 | the reference registry (events, derivation, redirects and register, settlement, corrections), the Daml Finance adapter, the twenty-scenario suite, the TypeScript library and CLI |
| 4 | 8 | integration support for the independent registry and the wallet |
| 5 | 8 | the DevNet or TestNet demonstration with a custodian application on its own participant, the guides and reference architecture |
| across | 8 | project management and Token Standards SIG coordination |

The reference registry is a reference and test implementation: the suite runs against it and adopters read it. It is not a production registry and is not audited under this grant. A registry that runs servicing in production audits its own implementation, as it would its token.

**Funded once.** If #878's default token has shipped by the start of Milestone 3, the reference registry uses it as its holding layer instead of building one. That saves about four engineer-weeks, and Milestone 3 is then reduced by 120,000 CC, those weeks at the grant's average rate.

If a milestone is missed:

- **The Milestone 1 gate fails.** The fallback design restricts the redirect's beneficiary to a provider-owned account, Milestone 5 drops the cross-provider case, Milestone 5 becomes 400,000 CC, and the total becomes 2,200,000 CC.
- **Milestone 4 has no independent adopter by week 24.** 300,000 CC of the 450,000 is not paid. The remaining 150,000 covers the milestone's integration support offered to adopters and its documented outreach, which are delivered either way. We name no adopter today and have approached none, so this risk is ours.
- **The Milestone 5 demonstration includes no application that is neither operated nor authored by PixelPlex.** 300,000 CC of Milestone 5 is not paid. A demonstration between PixelPlex's own applications does not show that the standard works across vendors.

The two Milestone 5 reductions do not compound: if the gate has already reduced Milestone 5 to 400,000 CC, a demonstration without a third-party application reduces it to 300,000 CC and no further.

Milestones 4 and 5 total 1,050,000 CC, of which **600,000 (25% of the grant, or 26% if the #878 reduction applies) depends on adoption by someone other than PixelPlex**, as described above. The remaining 450,000 covers integration support, outreach, the demonstration, documentation and handover, which are delivered either way. Milestones 4 and 5 are priced above their engineering effort; the difference is the adoption contingency and is paid only if someone else adopts.

Under the gate fallback, 300,000 of Milestone 4 and 100,000 of Milestone 5 (400,000 of 2,200,000, about 18%) depend on adoption.

### Volatility Stipulation
The project runs longer than 6 months, and the repository states that such projects may be re-evaluated for price volatility. If the USD/CC rate moves materially before the later milestones, we ask the Committee to re-evaluate the remaining milestone amounts on that basis.

### Maintenance
PixelPlex commits to 12 months of maintenance after Milestone 5 at its own cost, covering **the current and the previous Splice release** at any point in those 12 months, which are the two releases the conformance suite pins in CI. This includes compatibility of the package, the reference registry and the suite with each new Splice release within 30 days of its release, security fixes, and issue triage within 5 business days. Older releases are not maintained. Extensions (holder elections, unit changes, partial redemption, residual allocation against a funded total, securities-lending equivalent payments, mergers, spin-offs and rights issues) will be proposed separately.

---

## Co-Marketing

Upon release, the implementing entity will collaborate with the Foundation on:

- Announcement coordination for each milestone
- A technical blog per milestone and a case study with the first independent adopter
- A workshop in the Token Standards / Asset Standards SIG and recorded training for the Foundation's materials
- Listing in the ecosystem directory alongside the token standard packages

---

## Motivation

Much of an institution's operational effort goes into servicing: getting the coupon to the right party on the right day, redeeming at maturity, keeping a record of what has been paid, and correcting terms that were announced wrong. The Foundation's 2026–2028 roadmap lists corporate actions and asset lifecycle events as an RFP area because this is where tokenized securities meet day-to-day operations.

The common failure case is an ordinary one. A bond is pledged as collateral, so the recorded holder is not the party owed the coupon, and the registry, the custodian, the collateral application and the wallet are four products from four vendors. The token standard tells all four who holds what. It does not tell them who gets paid, or whether they have been paid. Without a standard, each pair of applications builds a bilateral integration.

The expected users are registries that issue income-bearing instruments, custodians that provide accounts, collateral applications, and wallets that display positions. The interfaces are additive, so adoption can be gradual: a registry can announce events before anyone consumes them, and a wallet can display them before any collateral application redirects them.

---

## Rationale

**Why a standard and not an application.** A servicing product would serve its own users and recreate the bilateral integration problem one layer up. The value lies in independently built applications agreeing with each other. That is why the main milestone is two independent applications agreeing, and why a quarter of the funding (about a fifth under the gate fallback) depends on adoption by someone other than PixelPlex.

**Why build on CIP-0112.** V2 already provides what this needs (accounts, custody relationships, mint and burn accounts) and is approved and funded. Building on it keeps a single ecosystem instead of splitting regulated assets into a separate one, which is also the motivation CIP-0112 states. It keeps this proposal small as well: five interfaces and their derivation rules, with no new asset model.

**Why a narrow version 1.** Coupons, redemption, a pledge and corrections are the smallest scope in which two independent applications can agree on every figure, including what has been paid. Each extension brings its own semantics to settle first. Elections need a way to disclose terms that a holder acts on before any entitlement exists. Unit changes depend on how a registry signs holdings. A securities loan is an obligation that remains after the borrower sells the bond, which a redirect on an account cannot express. Taking these on in this grant would put the core at risk for cases that not every registry has.

**Why entitlements are recorded objects.** Two parties that each compute the answer will eventually disagree, and the disagreement shows up weeks later as a payment break. A recorded entitlement, with its inputs and payments attached, turns reconciliation into a comparison.

**Why payment and record are one transaction.** A payment that carries a reference to what it pays helps a reconciler afterwards, but it does not prevent a second payment. Consuming the entitlement in the transaction that moves the money does prevent it, and it also orders a payment against a correction without extra coordination.

**Why the redirect is separate from the lock.** Extending `Lock` would require every registry to understand collateral and every collateral application to agree on one lock semantics, and it would change a V2 interface. A separate object attached to a position keeps the registry's task (pay the party the redirect names) independent of why the redirect exists.

**Why no new payment rail.** A distribution is a transfer. Registries, wallets and settlement venues already implement transfers and allocations, so a distribution that arrives as an ordinary transfer is one that existing software can already display and settle.

**Alternatives considered.** Extending Daml Finance would serve teams that already use it, but would not reach registries built directly on the token standard, where most assets are; the adapter covers both. Keeping servicing off-ledger, with only the payment on-ledger, was rejected because it leaves the entitlement, which is what parties disagree about, outside the record they share.

**About PixelPlex.** PixelPlex operates a Canton validator, four Featured Applications and an asset issuer, builds CC View (explorer and data API), Console Wallet and CC Tag (Canton Name Service front end), co-authored CIP-0103 (dApp Standard) and authored the draft Standard Package Validation CIP (cips PR #168) that the funded Package Distribution proposal builds on. About a third of its 100 engineers work on Canton and Daml. We operate both an issuer and a wallet on the same network, so we are on both sides of the record this standard asks two parties to agree on.
