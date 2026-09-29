# Token Publication Standard HTTP API

The CIP says what a publisher owes a client and what a successful identity check means. This file says how those calls look: paths, fields, and errors.

An archived identity stays on the catalog until the publisher removes it. A migration row stays only while the publisher still wants it read. More than one package may be `current`. Validate reports only the ids sent in the request. Holder and history reads stay required routes. The publisher may narrow who sees them. The other routes stay open.

## What you already have before the first call

A client starts with a base URL someone gave it. This API does not provide a directory of base URLs.

```text
{origin}/api/token-standard/v0/registrars/{registrarParty}/registry
```

| Piece | Meaning |
| --- | --- |
| `{origin}` | Scheme and host of the publisher, for example `https://registry.example`. No trailing slash. |
| `{registrarParty}` | Party id of the registrar that signs the instrument-identity contract. The same string is `instrumentAdmin` on every token card from this prefix. |

Everything below is relative to that prefix. A full catalog URL looks like this:

```text
https://registry.example/api/token-standard/v0/registrars/example-issuer::1220abcd/registry/published/v0/tokens
```

`{cid}` in later paths is the contract id of the instrument-identity contract. It is not the free-text instrument id. Put it in the path percent-encoded. Two tokens can share an instrument id. They cannot share a contract id.

## Calls in the order a client makes them

The example registrar is `example-issuer::1220abcd`. The example token is the free-text id `EXAMPLE`. Its identity contract id is `00example-identity-cid`. These values are placeholders so the same id can be followed from one response to the next. They are not a live token.

1. Catalog. Learn which tokens this registrar publishes, and take `contractId`.
2. One token. Fetch that card again when the client already knows the cid.
3. Version. Read the current label and a note of what it changed.
4. Migrations. Read changes the client must act on before a date.
5. Package status. See which package ids are live, scheduled, or no longer served.
6. Packages. Get the DAR URLs and download the bytes.
7. Validate. Check the package ids from those bytes against the published set, then upload on the client's own node.
8. Proof. Take the disclosed identity contract and exercise its no-op choice on the client's own participant.

The same `{cid}` also has these reads. They are not steps in the order above. The publisher must serve them and may withhold them from some callers. See Restricted reads.

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
- This API never calls `prepare`, `execute`, or `submit-and-wait`. The client submits the identity-choice exercise on its own participant.
- Unknown JSON fields should be ignored, so a field can be added later without breaking a client.
- A missing required field is a failed response. Do not fill it in from a previous call.

### Errors

| Status | When | Body |
| --- | --- | --- |
| 200 | The call succeeded. A validate mismatch is still 200. See that section. | The resource body |
| 400 | The body is not JSON, or a required field is missing or has the wrong JSON type | `{ "error": "<message>" }` |
| 403 | A restricted read is not available to this caller. `error` is exactly `not_public`. Not used on any other path. | `{ "error": "not_public" }` |
| 404 | This registrar does not publish `{cid}` | `{ "error": "<message>" }` |
| 500 | Unexpected failure on the publisher | `{ "error": "<message>" }` |

`404` means the publisher has removed `{cid}` from this registrar's catalog. It does not mean the contract was archived. While the publisher still lists the cid, the catalog, `GET /published/v0/tokens/{cid}`, and proof all return it, including after the contract is archived on the ledger. The inactive-contract result happens later, on the client's participant, when the choice is exercised. After the publisher removes the cid, those three calls are `404`.

Transfer, allocation, and settlement are not part of this file.

## Restricted reads

These paths must exist for every published token. They are the only paths whose visibility the publisher may narrow.

| Path | Read |
| --- | --- |
| `/published/v0/tokens/{cid}/holdings` and any path under it | Holdings of the token |
| `/published/v0/tokens/{cid}/activities` | Activity on the token |
| `/published/v0/tokens/{cid}/holders` | Holders of the token |
| `/published/v0/tokens/{cid}/updates` and any path under it | Transaction history |

The route is required even when the publisher does not want the data public. Withholding is `403` with `error` exactly `not_public`. A missing route means the server does not implement the read. `404` means this registrar does not publish `{cid}`. An empty `200` means there is nothing to report.

The body of a successful response is not defined here. The access rule is. No other path in this file may return `not_public`.

### Access token

Send the secret on these paths only:

```http
Authorization: Bearer <token>
```

The token is an opaque string the publisher gave the caller. Other paths in this file do not read this header. Sending it on a catalog, package, version, migration, or proof call does not change that call.

If the publisher serves the read to everyone, the header may be omitted.

If the publisher does not serve the read to this caller, the response is `403` whether the header is missing, wrong, or the publisher serves the route to nobody:

```json
{ "error": "not_public" }
```

`error` is that exact string. `404` is not used for a withheld read. `404` still means this registrar does not publish `{cid}`. An empty `200` is not a substitute for `not_public`. Empty means there is nothing to report.

## Catalog

`GET /published/v0/tokens`

Use this when the client knows the registrar and needs the contract id. The free-text instrument id is on the card so the client can match a name it already has. After this call, every other path uses `contractId`.

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
      "contractId": "00example-identity-cid",
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
| `tokens[].contractId` | Identity contract id. This is `{cid}` for every other resource. |
| `tokens[].instrumentId` | Free-text instrument id. Not unique. Not `{cid}`. |
| `tokens[].instrumentAdmin` | Registrar party. Equals `{registrarParty}` in the URL. |
| `nextPageToken` | Pass this as `pageToken` to get the next page. `null` means this was the last page. |

The list order is not significant. Do not treat the first card as the newest token.

`instrumentAdmin` on a card from this prefix is always the party in the URL. A token signed by some other party does not appear here.

### One token

`GET /published/v0/tokens/{cid}`

Same card, without the `tokens` wrapper. Use it when the client stored the cid earlier and wants to confirm the registrar still publishes it.

```json
{
  "contractId": "00example-identity-cid",
  "instrumentId": "EXAMPLE",
  "instrumentAdmin": "example-issuer::1220abcd"
}
```

`404` if the publisher has removed that cid from this registrar's catalog. An archived contract that the publisher still lists is `200` with the same card.

## Version

`GET /published/v0/tokens/00example-identity-cid/version`

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

`GET /published/v0/tokens/00example-identity-cid/migrations`

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

`GET /published/v0/tokens/00example-identity-cid/package-updates`

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

More than one row may be `current`. The identity package and the token package are the usual case. The set of `packageId` values with `status` `current` is the same set as `GET .../packages`.

`planned` is only an announcement: when the publisher starts serving those bytes, the id shows up on the package list and becomes `current`. A `deprecated` id is absent from that list.

## Packages

`GET /published/v0/tokens/00example-identity-cid/packages`

Every package the token's contracts need, including the small package that defines the instrument-identity contract. Download these before validate.

```json
{
  "packages": [
    {
      "packageId": "aaa111",
      "url": "https://cdn.example/usd-identity.dar"
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
| `url` | Absolute URL of the DAR that contains that package. Fetching it may return a redirect. Follow the redirect. The DAR bytes are not in this JSON. |

The list is not empty for a published token. One URL may be repeated when a single DAR contains several of the packages. `aaa111` in the example is the identity package. The same id comes back as `packageId` on the proof.

The client reviews the DAR before vetting it. This endpoint does not review it.

## Validate

`POST /published/v0/tokens/00example-identity-cid/packages/validate`

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

`GET /published/v0/tokens/00example-identity-cid/proof`

The disclosed instrument-identity contract, plus the package id the client audits. This response is the input to an exercise on the client's participant. It is not itself the check. A `200` from this endpoint does not mean the contract is real.

```json
{
  "contractId": "00example-identity-cid",
  "templateId": "aaa111:Token.Identity:InstrumentIdentity",
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

The choice name is not in this JSON. The client reads the single no-op choice from the package `packageId` after auditing it. The CIP fixes the behavior of that choice (non-consuming, no ledger effect), not its name.

### What the client does with the proof

1. Download the DAR whose package list entry has this `packageId`.
2. Read the package. It contains the identity contract and one choice that does not create, archive, or update anything.
3. `POST .../packages/validate` with that package id. Continue only if `matches` is `true`.
4. Vet the package on the client's own participant.
5. Exercise the no-op choice there, as the client's own party, with the proof fields copied into `disclosedContracts`.

The exercise argument is whatever that package defines for its one choice. The publication server is not called again to grade the result.

```json
{
  "commands": [
    {
      "ExerciseCommand": {
        "templateId": "aaa111:Token.Identity:InstrumentIdentity",
        "contractId": "00example-identity-cid",
        "choice": "<name read from the audited package>",
        "choiceArgument": {
          "actor": "wallet-backend::1220ffff"
        }
      }
    }
  ],
  "disclosedContracts": [
    {
      "templateId": "aaa111:Token.Identity:InstrumentIdentity",
      "contractId": "00example-identity-cid",
      "createdEventBlob": "<base64>",
      "synchronizerId": "global-domain::1220sync"
    }
  ]
}
```

`actor` is the party the client submits as. The field name inside `choiceArgument` comes from the package, as does the choice name. `wallet-backend::1220ffff` is not the registrar.

| Exercise result | What the client concludes |
| --- | --- |
| Success | The contract was created under `instrumentAdmin`. The client may treat `00example-identity-cid` as this token's id. |
| Any other failure | The client does not treat the token as verified. |
| Inactive contract | The id once referred to a contract, and that contract is no longer active. This is not a successful check. |

The publisher should not archive the identity contract. Archiving it does not break transfer or settlement. It only makes this exercise fail.
