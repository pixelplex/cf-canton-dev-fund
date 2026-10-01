# Token Publication Standard HTTP API

This document is the normative HTTP specification for version 0 of the Token Publication Standard. It is not an implementation. The CIP states what a publisher owes a client and what a successful registration check means. This document states the paths, fields, status codes, and errors. The two documents are one specification: the CIP wins on meaning, and this document wins on the JSON shape. A breaking change to either document is a new version of both.

An archived registration stays on the catalog until the publisher removes it. A migration row stays only while the publisher still wants it read. More than one package may be `current`. Validate reports only the ids sent in the request. Holder and history reads stay required routes. The publisher may narrow who sees them. The other routes stay open.

## What you already have before the first call

A client starts with a base URL someone gave it. This API does not provide a directory of base URLs.

```text
{origin}/api/token-standard/v0/registrars/{registrarParty}/registry
```

| Piece | Meaning |
| --- | --- |
| `{origin}` | Scheme and host of the publisher, for example `https://registry.example`. No trailing slash. |
| `{registrarParty}` | Party id of the registrar that signs the publication-registration contract. The same string is `instrumentAdmin` on every token card from this prefix. |

Everything below is relative to that prefix. A full catalog URL looks like this:

```text
https://registry.example/api/token-standard/v0/registrars/example-issuer::1220abcd/registry/published/v0/tokens
```

`{cid}` in later paths is the publication id: the contract id of the publication-registration contract. Put it in the path percent-encoded. This API keys each published token by that contract id. Holdings, transfer, allocation, and settlement keep their own instrument identifiers. `instrumentId` on a card is the instrument id this publication refers to. It is not `{cid}`.

## Calls in the order a client makes them

The example registrar is `example-issuer::1220abcd`. The example token is the free-text id `EXAMPLE`. Its publication-registration contract id is `00example-registration-cid`. These values are placeholders so the same id can be followed from one response to the next. They are not a live token.

1. Catalog. Learn which tokens this registrar publishes, and take `contractId`.
2. One token. Fetch that card again when the client already knows the cid.
3. Version. Read the current label and a note of what it changed.
4. Migrations. Read changes the client must act on before a date.
5. Package status. See which package ids are live, scheduled, or no longer served.
6. Packages. Get the DAR URLs and download the bytes.
7. Validate. Check the package ids from those bytes against the published set, then upload on the client's own node.
8. Proof. Take the disclosed publication-registration contract and exercise its no-op choice on the client's own participant.

The same `{cid}` also has these reads. They are not steps in the order above. The routes must exist and the server must answer them. The publisher chooses, for each read, to serve it, not to serve it, or to refuse it and name a URL where the caller can obtain a token. See Restricted reads.

- `/published/v0/tokens/{cid}/holdings` and any path under it. Holdings of the token.
- `/published/v0/tokens/{cid}/activities`. Activity on the token.
- `/published/v0/tokens/{cid}/holders`. Holders of the token.
- `/published/v0/tokens/{cid}/updates` and any path under it. Transaction history.

Version, migrations, and package status answer different questions. Calling one does not replace the others.

| Call | Question it answers |
| --- | --- |
| Version | What label is the token on right now, and what did that label change? |
| Migrations | Must I change my integration before a date, and what exactly do I change? |
| Package status | Which package id should I be vetting, which one is coming, which one is gone? |

A new required key in a free-form argument map shows up in migrations. It does not need a new package id. A package replacement shows up in package status. The version note is the human summary of the label the publisher is on today.

## Conventions

- Request and response bodies are JSON objects. `Content-Type: application/json`.
- Catalog, version, migrations, package status, packages, validate, and proof need no access token. Possession of the base URL is enough.
- Holder and history reads may require an access token. See Restricted reads. No other path uses it.
- GET calls only read. They do not change publisher state.
- `POST .../packages/validate` only compares ids. It does not accept DAR bytes, vet a package, or submit a ledger command.
- This API never calls `prepare`, `execute`, or `submit-and-wait`. The client submits the registration-choice exercise on its own participant.
- Unknown JSON fields should be ignored, so a field can be added later without breaking a client.
- A missing required field is a failed response. Do not fill it in from a previous call.

### Errors

| Status | When | Body |
| --- | --- | --- |
| 200 | The call succeeded. A validate mismatch is still 200. See that section. | The resource body |
| 400 | The body is not JSON, or a required field is missing or has the wrong JSON type | `{ "error": "<message>" }` |
| 403 | This caller is not served a restricted read. Not used on any other path. | See Restricted reads |
| 404 | This registrar does not publish `{cid}` | `{ "error": "<message>" }` |
| 500 | Unexpected failure on the publisher | `{ "error": "<message>" }` |

`404` means the publisher has removed `{cid}` from this registrar's catalog. It does not mean the contract was archived. While the publisher still lists the cid, the catalog, `GET /published/v0/tokens/{cid}`, and proof all return it, including after the contract is archived on the ledger. The inactive-contract result happens later, on the client's participant, when the choice is exercised. After the publisher removes the cid, those three calls are `404`.

Transfer, allocation, and settlement are not part of this file.

## Restricted reads

These paths must exist for every published token, and the server must answer them. They are the only paths the publisher may refuse to serve.

| Path | Read |
| --- | --- |
| `/published/v0/tokens/{cid}/holdings` and any path under it | Holdings of the token |
| `/published/v0/tokens/{cid}/activities` | Activity on the token |
| `/published/v0/tokens/{cid}/holders` | Holders of the token |
| `/published/v0/tokens/{cid}/updates` and any path under it | Transaction history |

For each read, the publisher chooses one of three. The route stays in place for all three.

| Choice | What the caller gets |
| --- | --- |
| Serve the read | The successful body. The publisher may still require a token it accepts for that caller. |
| Do not serve the read | `403` for every caller. `tokenUrl` is absent. |
| Serve it only with a token, and say where to get one | `403` with `tokenUrl` when the caller has no accepted token. The successful body when the token is accepted. |

A refusal is `403` with this body. `message` below is a sample. The publisher writes its own sentence. `tokenUrl` is included only for the third choice.

```json
{
  "error": "access_not_granted",
  "message": "The publisher does not serve this read to this caller.",
  "tokenUrl": "https://publisher.example/request-access"
}
```

| Field | Rule |
| --- | --- |
| `error` | Exactly `access_not_granted`. |
| `message` | A sentence from the publisher. The sentence in the sample is not required text. It must say that this caller is not served the read. |
| `tokenUrl` | Optional. An absolute URL where the caller can obtain a token. Omit the field when the publisher has chosen not to serve the read to anyone. |

A missing route means the server does not implement the read. `404` means this registrar does not publish `{cid}`. An empty `200` means the caller is served and there is nothing to report. The publisher must not use an empty `200` to mean "not served".

The body of a successful response is not defined here. The access rule is. No other path in this file may return `access_not_granted`.

### Access token

The token is an opaque string the publisher gave the caller. This API does not say how the client sends it. The client decides that.

Other paths in this file do not use the token. Sending one on a catalog, package, version, migration, or proof call does not change that call.

If the publisher serves the read to everyone, the caller needs no token.

If the publisher does not serve the read to this caller, the response is `403` with `access_not_granted`. That includes a missing token, a token the publisher does not accept, and a publisher that serves the read to nobody. `tokenUrl` is present only when the publisher chose to name where a token can be obtained.

`404` is not used for a read the publisher does not serve. `404` still means this registrar does not publish `{cid}`. An empty `200` is not a substitute for `access_not_granted`. Empty means the caller is served and there is nothing to report.

## Catalog

`GET /published/v0/tokens`

Use this when the client knows the registrar and needs the publication contract id. `instrumentId` is on the card so the client can match the instrument id it already uses for holdings or transfer. After this call, every path in this API uses `contractId`.

### Query

| Name | Required | Meaning |
| --- | --- | --- |
| `pageSize` | no | Maximum number of cards in this page. The server may return fewer. |
| `pageToken` | no | The `nextPageToken` from the previous page. Omit it on the first page. |

### Response

```json
{
  "tokens": [
    {
      "contractId": "00example-registration-cid",
      "instrumentId": "EXAMPLE",
      "instrumentAdmin": "example-issuer::1220abcd"
    }
  ],
  "nextPageToken": null
}
```

| Field | Meaning |
| --- | --- |
| `tokens` | Cards on this page. May be empty when the registrar publishes nothing. |
| `tokens[].contractId` | Publication-registration contract id. This is `{cid}` for every other resource. |
| `tokens[].instrumentId` | Instrument id this publication refers to. For a CIP-0056 token, that standard's instrument id. Not `{cid}`. |
| `tokens[].instrumentAdmin` | Registrar party. Equals `{registrarParty}` in the URL. |
| `nextPageToken` | Pass this as `pageToken` to get the next page. `null` means this was the last page. |

The list order is not significant. Do not treat the first card as the newest token.

`instrumentAdmin` on a card from this prefix is always the party in the URL. A token signed by some other party does not appear here.

### One token

`GET /published/v0/tokens/{cid}`

Same card, without the `tokens` wrapper. Use it when the client stored the cid earlier and wants to confirm the registrar still publishes it.

```json
{
  "contractId": "00example-registration-cid",
  "instrumentId": "EXAMPLE",
  "instrumentAdmin": "example-issuer::1220abcd"
}
```

`404` if the publisher has removed that cid from this registrar's catalog. An archived contract that the publisher still lists is `200` with the same card.

## Version

`GET /published/v0/tokens/00example-registration-cid/version`

The publisher's current label for the whole token, plus a human note of what that label changed. This is not a package id and not a sort key. Do not decide that `2.4.0` is newer than `2.10.0` by parsing the string. If the client must change behavior, that fact is in migrations.

```json
{
  "version": "2.3.0",
  "description": "adds a required settlementRef key to ExtraArgs.meta"
}
```

| Field | Meaning |
| --- | --- |
| `version` | Non-empty label. A later migration's `targetVersion` uses the same kind of label. |
| `description` | Non-empty text. What changed in this label, written for an integrator. |

Both fields are always present. A publisher that has nothing finer to say still returns a description, for example `"initial publication"`.

## Migrations

`GET /published/v0/tokens/00example-registration-cid/migrations`

Announcements the integrator may have to act on. The list is the full set, not a page. An empty array means there is nothing outstanding. It does not mean the call failed.

```json
{
  "migrations": [
    {
      "requiresAction": true,
      "targetVersion": "3.0.0",
      "effectiveAt": "2026-11-01T00:00:00Z",
      "description": "settlementRef becomes mandatory; unset values will be rejected after this date"
    }
  ]
}
```

| Field | Meaning |
| --- | --- |
| `requiresAction` | `true`: a client that keeps the old behavior will be rejected after `effectiveAt`. `false`: the note is informational, the old call still works. |
| `targetVersion` | Version label the publisher intends to serve from `effectiveAt`. Compare it to `version` only as text equality, not as a number. |
| `effectiveAt` | RFC 3339 date-time in UTC. The moment the announcement takes effect. |
| `description` | What the integrator changes. This is the sentence that used to be a direct message to each backend team. |

Every field is required on every element. A typical `requiresAction: true` case is a new required key in a free-form argument map. That change often has no new package, so it will not appear as a `planned` row in package status.

The array is the announcements still in force, not a history log. The publisher removes a row when that announcement no longer needs to be read. A client that sees a row acts on it whether or not `effectiveAt` is in the past. A client that no longer sees a row it saw earlier treats the announcement as withdrawn. The label the publisher is on now is `version`.

The client should poll this. The publisher does not push.

## Package status

`GET /published/v0/tokens/00example-registration-cid/package-updates`

One row per package the publisher wants the client to know about. This includes packages that are not downloadable yet (`planned`) and packages that used to be served (`deprecated`).

```json
{
  "packages": [
    {
      "packageId": "bbb222",
      "status": "current"
    },
    {
      "packageId": "ccc333",
      "status": "planned",
      "effectiveAt": "2026-11-01T00:00:00Z",
      "message": "please vet before this date"
    },
    {
      "packageId": "ddd444",
      "status": "deprecated"
    }
  ]
}
```

| `status` | What `packageId` is | `effectiveAt` | `message` |
| --- | --- | --- | --- |
| `current` | The package the publisher serves now. It is on the package list. | Omitted | Omitted |
| `planned` | A future package id. The bytes are not necessarily on the package list yet. | Required. RFC 3339 UTC. The date by which the publisher asks clients to be ready. | Optional. Human text, for example `please vet before this date`. |
| `deprecated` | A package the publisher no longer serves. It is absent from the package list. | Optional | Optional |

A `status` other than these three should be ignored. Do not upload that package on the strength of the row.

More than one row may be `current`. The registration package and the token package are the usual case. The set of `packageId` values with `status` `current` is the same set as `GET .../packages`.

`planned` is only an announcement: when the publisher starts serving those bytes, the id shows up on the package list and becomes `current`. A planned row that is gone, and whose package id never became `current`, is a withdrawn plan. A replacement is a new `planned` row in place of the old one. There is no cancelled status. A `deprecated` id is absent from that list.

## Packages

`GET /published/v0/tokens/00example-registration-cid/packages`

The package ids the client has to obtain from this publication, including the small package that defines the publication-registration contract. The list does not include the transitive packages of Splice or the Token Standard. Each `url` is a DAR archive: the downloaded bytes are that archive, not a package extracted from it. Download these before validate.

```json
{
  "packages": [
    {
      "packageId": "aaa111",
      "url": "https://cdn.example/usd-registration.dar"
    },
    {
      "packageId": "bbb222",
      "url": "https://cdn.example/usd-token.dar"
    }
  ]
}
```

| Field | Meaning |
| --- | --- |
| `packageId` | Canton package id. This is the hash of the package, not a hash of the whole DAR file. |
| `url` | Absolute URL of the DAR archive that contains that package. The downloaded body is the archive. Fetching it may return a redirect. Follow the redirect. The DAR bytes are not in this JSON. |

The list is not empty for a published token. One URL may be repeated when a single DAR contains several of the packages. `aaa111` in the example is the registration package. The same id comes back as `packageId` on the proof.

The client reviews the DAR before vetting it. A published URL and package id do not mean that the client or the network has audited the package or vetted it. This endpoint does not review it. The client decides whether to vet it.

## Validate

`POST /published/v0/tokens/00example-registration-cid/packages/validate`

Call this after the client has downloaded the DARs and computed each Canton package id from the bytes, and before it vets those packages on its node.

The request body is the ids the client computed. It is not the file.

```json
{
  "packageIds": ["aaa111", "bbb222", "ffff999"]
}
```

`packageIds` is required and is a non-empty array of strings. An empty array, a missing field, or a non-array is `400`.

```json
{
  "results": [
    { "packageId": "aaa111", "matches": true },
    { "packageId": "bbb222", "matches": true },
    { "packageId": "ffff999", "matches": false }
  ]
}
```

| Field | Meaning |
| --- | --- |
| `results` | One element per requested id, in the same order. |
| `results[].packageId` | The id from the request. |
| `results[].matches` | `true` only when that id is in the package list for this token. |

A mismatch is `matches: false` with status 200. `400` is only a bad body. `404` is only an unknown token cid.

`results` contains one element per id in the request, and no others. An id that is on the package list but missing from `packageIds` does not appear. The client checks that it asked about the whole live set by comparing `GET .../packages` to the ids it computed from the bytes. `matches: true` on every sent id does not mean the request contained the whole set.

The client vets an id only when `matches` is `true`. `ffff999` in the example was not published for `EXAMPLE`, so the client does not upload it.

`planned` ids are not on the package list yet, so validate returns `matches: false` for them until the publisher starts serving the bytes. That is not a signal to upload them early.

## Proof

`GET /published/v0/tokens/00example-registration-cid/proof`

The disclosed publication-registration contract, plus the package id the client audits. This response is the input to an exercise on the client's participant. It is not itself the check. A `200` from this endpoint does not mean the contract is real.

```json
{
  "contractId": "00example-registration-cid",
  "templateId": "aaa111:Token.Publication:PublicationRegistration",
  "createdEventBlob": "<base64>",
  "synchronizerId": "global-domain::1220sync",
  "packageId": "aaa111"
}
```

| Field | Meaning |
| --- | --- |
| `contractId` | Same as `{cid}` and as `contractId` on the catalog card. |
| `templateId` | `<package-id>:<module>:<template name>`. The package-id segment equals `packageId`. |
| `createdEventBlob` | Base64 disclosed-contract blob. Pass it through on the exercise. Do not decode it to decide whether the contract is valid. |
| `synchronizerId` | Synchronizer the contract is assigned to. The client submits the exercise there. |
| `packageId` | Package to audit and vet before the exercise. It also appears in the package list. |

The choice name is not in this JSON. It is `PublicationRegistration_Ping`, fixed by the CIP. The choice is non-consuming and has no ledger effect.

### What the client does with the proof

1. Download the DAR whose package list entry has this `packageId`.
2. Read the package. It contains the publication-registration contract and one choice, `PublicationRegistration_Ping`, that does not create, archive, or update anything.
3. `POST .../packages/validate` with that package id. Continue only if `matches` is `true`.
4. Vet the package on the client's own participant.
5. Exercise the no-op choice there, as the client's own party, with the proof fields copied into `disclosedContracts`.

The choice argument is `actor`, the party the client submits as. The publication server is not called again to grade the result.

```json
{
  "commands": [
    {
      "ExerciseCommand": {
        "templateId": "aaa111:Token.Publication:PublicationRegistration",
        "contractId": "00example-registration-cid",
        "choice": "PublicationRegistration_Ping",
        "choiceArgument": {
          "actor": "wallet-backend::1220ffff"
        }
      }
    }
  ],
  "disclosedContracts": [
    {
      "templateId": "aaa111:Token.Publication:PublicationRegistration",
      "contractId": "00example-registration-cid",
      "createdEventBlob": "<base64>",
      "synchronizerId": "global-domain::1220sync"
    }
  ]
}
```

`actor` is the party the client submits as. `wallet-backend::1220ffff` is not the registrar.

| Exercise result | What the client concludes |
| --- | --- |
| Success | The contract was created under `instrumentAdmin`. The client may treat `00example-registration-cid` as this published token's id in this API. Holdings, transfer, allocation, and settlement keep their own instrument identifiers. |
| Any other failure | The client does not treat the token as verified. |
| Inactive contract | The id once referred to a contract, and that contract is no longer active. This is not a successful check. |

The publisher should not archive the publication-registration contract. Archiving it does not break transfer or settlement. It only makes this exercise fail.
