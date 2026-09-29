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

The Token Publication Standard gives an integrating backend three facts that today arrive only by personal message: which packages the token needs, when a breaking change is coming, and which token it actually is.

The identity is the contract id of a small instrument-identity contract the token author creates when the token is published. The contract carries the token's admin and instrument id, and it exposes one choice that changes nothing. A backend audits that package, uploads it to its own participant, and exercises the choice on the disclosed contract. Canton checks the author's authorization as part of the exercise, so the result does not depend on trusting the server that returned the contract id.

Transfer, allocation, and settlement are unchanged. This CIP does not define a directory of registry URLs. Reads of who holds a token, and of its transaction history, are part of publication, and a publisher may withhold them. The other facts may not. The HTTP representation is a separate document.

## Copyright

This CIP is licensed under CC0-1.0: [Creative Commons CC0 1.0 Universal](https://creativecommons.org/publicdomain/zero/1.0/)

## Specification

The Token Publication Standard is the set of facts a publisher gives a client, and the on-ledger check a client uses to confirm identity.

A **publisher** is the author of a token's contracts and the operator of the node those contracts are deployed on. This CIP treats those as one role. A **client** is a backend that integrates the token. A **published token** is a token for which the publisher has created an instrument-identity contract and exposes the publication facts below.

### Publication facts

For each published token, the publisher exposes the packages that the token's contracts need, and how to get the binaries. The response MAY redirect to a blob or a CDN. Before uploading a package to its own node, a client checks the hash against this set. A client SHOULD audit each package it uploads.

The same publication carries the current version label and a description of what changed in it, and announcements of changes an integrator has to handle. Such a change need not be a new package. A new required key in a free-form argument map is the usual case. An announcement says whether action is required, which version it belongs to, when it takes effect, and what to change.

Each package is marked `current`, `planned`, or `deprecated`. A current package is the one served now. A planned one names a future package id and a date, and MAY include a short message. A deprecated package is one the publisher no longer serves.

The publication also carries the contract id of the instrument-identity contract, the disclosed contract, and the package that defines it.

The publication surface does not prepare, execute, or submit the end user's ledger transaction. That stays with the user or their application. The surface only supplies the facts above.

A publisher who does not opt in publishes nothing under this CIP and remains usable exactly as before. Opt-in is per token, advertised by a `supportedApis` flag that a client which does not know the Token Publication Standard ignores.

### Restricted reads

For each published token, the publisher MUST serve these reads. A publisher MAY withhold any of them from callers it does not want to serve. No other read in this CIP may be withheld.

- `/published/v0/tokens/{cid}/holdings/*`
- `/published/v0/tokens/{cid}/activities`
- `/published/v0/tokens/{cid}/holders`
- `/published/v0/tokens/{cid}/updates/*`

Withholding MUST be an explicit error that the read is not available to this caller. The publisher MUST NOT express that by leaving the route out, by answering as if the token were unknown, or by returning an empty result. An empty result means there is nothing to report. A missing route means the server does not implement the read.

### Access token

Only the reads in the previous section MAY require a token sent with the request. The token is an opaque secret the publisher gives to a caller it is willing to serve. This CIP does not say how that secret is issued or rotated.

A publisher that serves one of those reads to everyone MUST accept the call with no token. A publisher that withholds it MUST reject a call that does not present a token the publisher accepts, and MUST use the same explicit error as any other withholding of that read.

Packages, version, migrations, package status, and identity MUST be served with no token. A missing or rejected token on a restricted read MUST NOT affect those calls.

### Instrument identity

The publisher MUST create the instrument-identity contract before presenting the token as published. Its contract id is the token's unique id. A second contract is a second token, even when the free-text instrument id is the same.

The contract is signed by the registrar. It has two fields:

| Field | Rule |
| --- | --- |
| `instrumentId` | The token's free-text instrument id |
| `instrumentAdmin` | The registrar party, and the contract's signatory |

It has one choice, `InstrumentIdentity_Ping`. The choice is non-consuming and has no effect: it does not update fields, create contracts, or archive anything. Exercising it either succeeds or fails. A client exercises it so the client's own participant can confirm that this contract was created.

```daml
template InstrumentIdentity
  with
    instrumentId : Text
    instrumentAdmin : Party
  where
    signatory instrumentAdmin

    -- Exists so a client can exercise it and trust that this contract
    -- was actually created. The exercise changes nothing. Canton checks
    -- the signatory's authorization as part of a successful exercise.
    nonconsuming choice InstrumentIdentity_Ping : ()
      with
        actor : Party
      controller actor
      do pure ()
```

No other contract the token needs depends on this one. Archiving it MUST NOT be required to transfer, allocate, or settle the token. Publishers SHOULD NOT archive it. After archival, a later exercise fails because the contract is inactive. A client MAY treat that failure as evidence the contract id once existed, and MUST NOT treat it as a successful check.

### Independent verification

A disclosed contract id in the publication facts is still a claim by the publisher's server. A client SHOULD confirm it before treating the contract id as the token's identity:

1. Take the disclosed contract and the package that defines the instrument identity.
2. Audit that package. It is limited to this contract and its no-op choice, so the audit is a short review.
3. Upload the package to the client's own participant.
4. Exercise the no-op choice against the disclosed contract.

Canton checks package vetting, the signatory, and the signature over the original create. Success means the contract was created under the registrar's authority. The check does not go back through the server that issued the claim. Failure means the client MUST NOT treat the token as verified.

### Non-goals

This CIP does not replace transfer, allocation, or settlement, and it does not add a second factory for those operations.

This CIP does not standardize which base URLs exist. A client keeps its own list. Lists maintained by the community are expected, and they are not part of the standard.

How a publisher hosts the publication surface, in which language, and how custom argument construction is implemented, are not part of the standard.

## Motivation

A Canton token is not usable from outside the author's node unless a backend has the packages, knows how the token behaves, and can tell it from another token with the same name. Today the author's developers and the backend's developers arrange all three by personal message.

That fails in the same way for three groups. The author can write flexible contracts, and still have no standard way for an outside backend to see the packages those contracts need. The backend, for each new token and each later change, has to learn that token from scratch and often has to ask the author again. Users inherit the cost: applications support few tokens, and tokens get used in flows they were not built for.

The pair `(instrument admin, instrument id)` does not separate two tokens. The admin is a party and the id is free text, and both are chosen by the issuer. Nothing stops one or several admins from issuing different tokens under the same name. A backend can then treat the wrong contract as the right one because the name matched.

Transfer and allocation do not close this gap. They describe how to move a token the backend has already identified and already holds packages for.

## Rationale

### Contract id

A string next to the token is as easy to forge as the `(admin, id)` pair. A contract id exists only after a create signed by the signatory. The instrument-identity contract is that create. The publication record then points at its contract id.

### The no-op choice

A contract id taken from the HTTP response is still the server's claim. The identity package is small, so a client can read it. Exercising the no-op choice is what makes the client's own participant check the authorization chain.

The choice does nothing to the token, so the check stays separate from transfer rules and the package stays small enough to review. The choice is named `InstrumentIdentity_Ping`.

### Version, package status, and migrations

Package status says a package id is `current`, `planned`, or `deprecated`. It does not say what the integrator has to change. A new required key in a free-form argument map is not a new package id at all.

The version label is the publisher's name for the current behavior, with a note of what that label changed. A migration says an integrator has to act, by when, and how. Each of the three leaves a gap if the other two are missing: the package timeline, the label, or the instruction.

### Holdings and history

Packages, the version, and the contract id are enough to integrate a token. Holdings and transfer history are about holders. A publisher can keep those off the public internet and still be usable.

### Base URLs

A client that has the publisher's base URL can read the facts and run the check. Listing every base URL on the network is a different problem, the same kind as a public list of RPC endpoints. This CIP does not take it on.

### What stays out of the rules

A spec that also required one server layout would still leave every token author maintaining that server, which is a product question, not an interoperability question. Two publishers interoperate with one client when both expose the same facts and the same identity check. Hosting, process layout, and custom argument construction stay outside this text. The HTTP shapes are specified alongside this CIP, not inside it, for the same reason: the rules above are what a second implementation has to satisfy, and the paths can be read on their own.

### Alternatives considered

An off-ledger signed list of token ids was rejected. It moves the trust root off the ledger, which is the property the instrument-identity check is there to avoid.

Standardizing every free-form argument map, so that migrations would be unnecessary, was rejected. Tokens differ in those maps on purpose. The migration announcement tells every watching client about a change. It does not try to forbid the change.

## Examples

The parties and the instrument id below are placeholders. They are not a token on the network.

The publisher `example-issuer::1220abcd` creates an instrument-identity contract signed by that party, with `instrumentId` `EXAMPLE` and `instrumentAdmin` set to the same party. The contract id of that contract is the id clients use for `EXAMPLE` from then on.

A custody backend that integrates `EXAMPLE` watches the publication facts. It receives a migration whose description is that `settlementRef` becomes mandatory, with `requiresAction` set, a target version, and an effective time. It changes its argument construction before that time, instead of discovering the rejection in production and asking the publisher what changed.

Before it treats the contract id as `EXAMPLE`, the backend downloads the identity package, reads the no-op choice, vets the package on its own participant, and exercises the choice on the disclosed contract. The exercise succeeds. The backend does not ask the publication server to confirm the result.

## Backwards compatibility

The standard is additive and opt-in per token. Clients that never read the publication facts are unaffected. A publisher that does not advertise the `supportedApis` flag is not required to expose those facts, and existing clients keep working as they do now. Clients that do not recognize the flag ignore it.

Changing the meaning of the instrument-identity check, or dropping a publication fact listed above, is a breaking change.

## Reference implementation

This CIP does not include a server, a Daml package, or tests. The HTTP representation of the publication facts is specified in [token-publication-standart-http-api-impleementation.md](token-publication-standart-http-api-impleementation.md). A reference server, tests, and the identity package are required before this CIP can move to Final. They are not part of this text. A later package or API file MUST match the rules in Specification.
