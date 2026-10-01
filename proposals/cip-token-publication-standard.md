<pre>
  CIP: TBD
  Layer: Applications
  Title: Token Publication Standard
  Author: PixelPlex Inc. (Nikita Gerasimenok, Vladislav Demidovich)
  Status: Draft
  Type: Standards Track
  Created: 2026-09-24
  License: CC0-1.0
  Requires: 0056
</pre>

## Abstract

The Token Publication Standard gives an integrating backend three facts that today arrive only by personal message: which packages the token needs, when a breaking change is coming, and which published token those facts belong to.

Inside this CIP, a published token is the contract id of a small publication-registration contract the token author creates when the token is published. One contract is one published token. The contract carries the token's admin and instrument id, and it exposes one choice that changes nothing. A backend audits that package, uploads it to its own participant, and exercises the choice on the disclosed contract. Canton checks the author's authorization as part of the exercise, so the result does not depend on trusting the server that returned the contract id.

That contract id is the identifier this standard adds. The same id names the published token on a utility registry surface and on a CIP-0056 registry surface. Holdings, transfer, allocation, and settlement keep the instrument identifiers they already use. Those flows are otherwise unchanged. This CIP does not define a directory of registry URLs. Reads of who holds a token, and of its transaction history, are part of publication, and a publisher may withhold them. The other facts may not. The HTTP API is a separate normative document, [token-publication-http-api.md](token-publication-http-api.md).

## Copyright

This CIP is licensed under CC0-1.0: [Creative Commons CC0 1.0 Universal](https://creativecommons.org/publicdomain/zero/1.0/)

## Specification

The Token Publication Standard is the set of facts a publisher gives a client, and the on-ledger check a client uses to confirm the publication id.

A **publisher** is the author of a token's contracts and the operator of the node those contracts are deployed on. This CIP treats those as one role. A **client** is a backend that integrates the token. A **published token** is a token for which the publisher has created an publication-registration contract and exposes the publication facts below.

### Publication facts

For each published token, the publisher exposes the packages the client has to obtain from this publication, and the DAR archives that contain them. The list is those package ids. It does not include the transitive packages of Splice or the Token Standard. The response MAY redirect to a DAR on a blob store or a CDN. Before uploading a package to its own node, a client checks the package id against this set. A published URL and package id do not mean that the client or the network has audited the package or vetted it. The client decides whether to vet it. A client SHOULD audit each package it uploads.

The same publication carries the current version label and a description of what changed in it, and announcements of changes an integrator has to handle. Such a change need not be a new package. A new required key in a free-form argument map is the usual case. An announcement says whether action is required, which version it belongs to, when it takes effect, and what to change.

Each package is marked `current`, `planned`, or `deprecated`. A current package is one served now. A planned one names a future package id and a date, and MAY include a short message. A planned id that disappears before it is served as `current` is withdrawn. Replacing a plan removes that row and adds another planned row. There is no separate cancelled status. A deprecated package is one the publisher no longer serves.

The publication also carries the contract id of the publication-registration contract, the disclosed contract, and the package that defines it.

The publication surface does not prepare, execute, or submit the end user's ledger transaction. That stays with the user or their application. The surface only supplies the facts above.

A publisher who does not opt in publishes nothing under this CIP and remains usable exactly as before. Opt-in is per token, advertised by a `supportedApis` flag that a client which does not know the Token Publication Standard ignores.

### Restricted reads

These routes MUST exist for every published token, and the server MUST answer them. For each read, the publisher chooses one of three:

1. Serve it. A caller with access receives the successful body. The publisher MAY require a bearer token it accepts for that caller.
2. Not serve it. Every caller receives `403` and `access_not_granted`. The body has no `tokenUrl`.
3. Not serve it until the caller has a token, and say where to get one. A caller without an accepted token receives `403` and `access_not_granted`, and the body includes `tokenUrl`. A caller with an accepted token receives the successful body.

The reads are:

- `/published/v0/tokens/{cid}/holdings/*`
- `/published/v0/tokens/{cid}/activities`
- `/published/v0/tokens/{cid}/holders`
- `/published/v0/tokens/{cid}/updates/*`

No other read in this CIP may be withheld. The publisher MUST NOT express "not served" by leaving the route out, by answering as if the token were unknown, or by returning an empty result. An empty result means the caller is allowed to read and there is nothing to report. The error shape, including the sample message, is specified in the HTTP API.

### Access token

Only the reads in the previous section MAY require a token sent with the request. The token is an opaque secret the publisher gives to a caller it is willing to serve. This CIP does not say how that secret is issued or rotated, except that choice 3 above MAY name a `tokenUrl` on the error.

A publisher that serves one of those reads to everyone MUST accept the call with no token. A publisher that does not serve it to this caller MUST reject the call with `access_not_granted`. That is the same error when the token is missing, when it is wrong, and when the publisher serves the read to nobody.

Packages, version, migrations, package status, and the publication registration MUST be served with no token. A missing or rejected token on a restricted read MUST NOT affect those calls.

### Publication registration

A publication registration is the on-ledger record that this registrar publishes this instrument under this CIP. The publisher MUST create one publication-registration contract for each token it publishes under this CIP. The contract id is that published token's id in this standard. A second contract is a second published token here, even when the text in `instrumentId` is the same.

This id belongs only to this standard. A client uses it to name the published token whether that token is reached through a utility registry surface or through CIP-0056. A CIP-0056 instrument remains the pair `(instrumentAdmin, instrumentId)` from CIP-0056. Holdings, transfer, allocation, and settlement do not store this contract id and do not require the contract.

The contract is signed by the registrar. It has two fields:

| Field | Rule |
| --- | --- |
| `instrumentId` | The instrument id this publication refers to. For a CIP-0056 token, the instrument id from that standard |
| `instrumentAdmin` | The registrar party, and the contract's signatory. For a CIP-0056 token, the instrument admin from that standard |

It has one choice, `PublicationRegistration_Ping`. The choice is non-consuming and has no effect: it does not update fields, create contracts, or archive anything. Exercising it either succeeds or fails. A client exercises it so the client's own participant can confirm that this contract was created.

```daml
template PublicationRegistration
  with
    instrumentId : Text
    instrumentAdmin : Party
  where
    signatory instrumentAdmin

    -- Exists so a client can exercise it and trust that this contract
    -- was actually created. The exercise changes nothing. Canton checks
    -- the signatory's authorization as part of a successful exercise.
    nonconsuming choice PublicationRegistration_Ping : ()
      with
        actor : Party
      controller actor
      do pure ()
```

No other contract the token needs depends on this one. Archiving it MUST NOT be required to transfer, allocate, or settle the token. Publishers SHOULD NOT archive it. After archival, a later exercise fails because the contract is inactive. A client MAY treat that failure as evidence the contract id once existed, and MUST NOT treat it as a successful check.

### Independent verification

A disclosed contract id in the publication facts is still a claim by the publisher's server. A client SHOULD confirm it before treating the contract id as the published token's id in this CIP:

1. Take the disclosed contract and the package that defines the publication registration.
2. Audit that package. It is limited to this contract and its no-op choice, so the audit is a short review.
3. Upload the package to the client's own participant.
4. Exercise the no-op choice against the disclosed contract.

Canton checks package vetting, the signatory, and the signature over the original create. Success means the contract was created under the registrar's authority. The check does not go back through the server that issued the claim. Failure means the client MUST NOT treat the token as verified.

### HTTP API

Paths, JSON fields, status codes, and errors are specified in [token-publication-http-api.md](token-publication-http-api.md). That document is the normative HTTP API for version 0 of this CIP. It is not an implementation. This CIP wins on meaning. The HTTP API wins on the JSON shape. A breaking change to either document is a new version of both.

### Non-goals

This CIP does not replace transfer, allocation, or settlement, and it does not add a second factory for those operations. It does not replace the instrument identifiers those flows already use. The publication contract id is an identifier of this standard only.

This CIP does not standardize which base URLs exist. A client keeps its own list. Lists maintained by the community are expected, and they are not part of the standard.

How a publisher hosts the publication surface, in which language, and how custom argument construction is implemented, are not part of the standard.

## Motivation

A Canton token is not usable from outside the author's node unless a backend has the packages, knows how the token behaves, and can tell it from another token with the same name. Today the author's developers and the backend's developers arrange all three by personal message.

That fails in the same way for three groups. The author can write flexible contracts, and still have no standard way for an outside backend to see the packages those contracts need. The backend, for each new token and each later change, has to learn that token from scratch and often has to ask the author again. Users inherit the cost: applications support few tokens, and tokens get used in flows they were not built for.

CIP-0056 already identifies an instrument by `(instrument admin, instrument id)`. The text id is unique for one admin. This CIP leaves that pair in place. Holdings, transfer, allocation, and settlement do not gain a field for the publication contract id.

An integrator still has no publication id that means the same thing for a utility token and for a CIP-0056 token. The publication-registration contract is that id, and only inside this standard. The publication then attaches packages and announced changes to it. Transfer and allocation still describe how to move an instrument the backend has already identified. They do not say which packages that published token needs, or which change is coming.

## Rationale

### Contract id

Utility registrations and CIP-0056 tokens need one publication id in this standard, without a new id in holdings or transfer. A contract id is that publication id. It exists only after a create signed by the signatory. The publication-registration contract is that create. Clients of this CIP address the published token by that contract id. Clients of other standards keep the identifiers those standards already define.

### The no-op choice

A contract id taken from the HTTP response is still the server's claim. The registration package is small, so a client can read it. Exercising the no-op choice is what makes the client's own participant check the authorization chain.

The choice does nothing to the token, so the check stays separate from transfer rules and the package stays small enough to review. The choice is named `PublicationRegistration_Ping`.

### Version, package status, and migrations

Package status says a package id is `current`, `planned`, or `deprecated`. It does not say what the integrator has to change. A new required key in a free-form argument map is not a new package id at all.

The version label is the publisher's name for the current behavior, with a note of what that label changed. A migration says an integrator has to act, by when, and how. Each of the three leaves a gap if the other two are missing: the package timeline, the label, or the instruction.

### Holdings and history

Packages, the version, and the contract id are enough to integrate a token. Holdings and transfer history are about holders. A publisher can keep those off the public internet and still be usable.

### Base URLs

A client that has the publisher's base URL can read the facts and run the check. Listing every base URL on the network is a different problem, the same kind as a public list of RPC endpoints. This CIP does not take it on.

### What stays out of the rules

A spec that also required one server layout would still leave every token author maintaining that server, which is a product question, not an interoperability question. Two publishers interoperate with one client when both expose the same facts and the same registration check. Hosting, process layout, and custom argument construction stay outside this text. The HTTP API is specified alongside this CIP, in [token-publication-http-api.md](token-publication-http-api.md), for the same reason: the rules above are what a second implementation has to satisfy, and the paths can be read on their own.

### Alternatives considered

An off-ledger signed list of token ids was rejected. It moves the trust root off the ledger, which is the property the publication-registration check is there to avoid.

Standardizing every free-form argument map, so that migrations would be unnecessary, was rejected. Tokens differ in those maps on purpose. The migration announcement tells every watching client about a change. It does not try to forbid the change.

## Examples

The parties and the instrument id below are placeholders. They are not a token on the network.

The publisher `example-issuer::1220abcd` creates an publication-registration contract signed by that party, with `instrumentId` `EXAMPLE` and `instrumentAdmin` set to the same party. The contract id of that contract is the id clients of this CIP use for that published token. Holdings and transfer of a CIP-0056 token continue to use `instrumentAdmin` and `instrumentId`.

A custody backend that integrates `EXAMPLE` watches the publication facts. It receives a migration whose description is that `settlementRef` becomes mandatory, with `requiresAction` set, a target version, and an effective time. It changes its argument construction before that time, instead of discovering the rejection in production and asking the publisher what changed.

Before it treats the contract id as this published token, the backend downloads the registration package, reads the no-op choice, vets the package on its own participant, and exercises the choice on the disclosed contract. The exercise succeeds. The backend does not ask the publication server to confirm the result. It still addresses holdings and transfer with `instrumentAdmin` and `instrumentId`.

## Backwards compatibility

The standard is additive and opt-in per token. Existing instrument identifiers are unchanged. Clients that never read the publication facts are unaffected. A publisher that does not advertise the `supportedApis` flag is not required to expose those facts, and existing clients keep working as they do now. Clients that do not recognize the flag ignore it.

Changing the meaning of the publication-registration check, or dropping a publication fact listed above, is a breaking change.

## Reference implementation

A reference server, tests, and the registration package are not part of this text. They are required before this CIP can move to Final. A later server and package MUST match Specification and [token-publication-http-api.md](token-publication-http-api.md).
