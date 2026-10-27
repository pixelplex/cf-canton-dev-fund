# Canton Asset Servicing: implementation draft

Coupons, redemptions and corrections on a tokenized security, including one that is pledged, across applications built by different vendors. Companion to the Development Fund proposal. 2026-10-27, PixelPlex.

Written against the token standard as shipped in Splice 0.8.3 (`token-standard/`, `daml/splice-api-token-burn-mint-v1/`) and CIP-0112.

**Build target.** All 24 token-standard `daml.yaml` files (23 under `token-standard/` plus `daml/splice-api-token-burn-mint-v1/`) are `sdk-version: 3.5.2` with `--target=2.1`. Nineteen of the 24 also carry `--explicit-serializable=yes`; the five without it are `allocation-instruction-v1`, `allocation-v1`, `holding-v1`, `metadata-v1` and `transfer-instruction-v1`; `allocation-request-v1`, also V1-generation, carries it. This package takes all three. The constraint decides the arithmetic in §5.2, and `Serializable` as a deriving clause does not exist before SDK 3.5.2.

Code blocks below compile at that target, in two packages: the interfaces in one, and the registry-side template of §4.3 with the §2 helper and the §5.2 expression in another (§4.3 explains why they cannot share a package). Both packages exist as PixelPlex's working prototype, together with a script asserting the §5.4 rounding examples, and become the starting point of Milestone 2's public repository.

## 1. Goal and non-goals

**Goal.** A registry announces that an instrument pays or redeems, derives who is owed what as of a record time, and pays it. A custodian, a collateral application and a wallet built by three other vendors read all three from the ledger without a bilateral integration. The case it must handle is a pledged bond, where the recorded holder is not the party the coupon is owed to.

**Version 1 covers** cash distributions (coupon, dividend, interest), full redemption, redirection of income to a pledgee, payment with a record of what has been paid, and corrections and cancellations before and after payment.

**Extensions beyond version 1.** Each is a separate package on top of this one, proposed separately (§13): holder elections, unit changes (splits, consolidations), partial redemption, allocating a rounding residual against a funded total, securities-lending equivalent payments, and cross-instrument corporate actions (mergers, spin-offs, rights issues). An extension reuses version 1's keys and leaves its interfaces as they are. Anything it needs beyond them, such as a funded total, a unit result or an option set, it carries in interfaces of its own, implemented alongside version 1's on the same contracts.

**Non-goals.** A new payment rail (distributions are ordinary transfers). Tax logic (withholding is a declared input). Valuation and risk parameters (#665). Liquidation mechanics (#232). Legal enforceability of any agreement a redirect points at.

## 2. Actors

| Actor | Is | Does |
| --- | --- | --- |
| Instrument admin | `InstrumentId.admin` (the registry) | announces events, derives entitlements, pays, corrects |
| Holder | account principal of an `Account` holding the instrument | owed the income, unless part of it is redirected |
| Account provider | `Account.provider`, where set (the custodian) | already sees its clients' holdings; sees their entitlements and every redirect on their positions |
| Beneficiary | the pledgee | receives redirected income |
| Paying agent | the admin, or a party the admin delegates to | pays and records the payment (§6); a stakeholder of every entitlement it settles |

**Account parties**, throughout, means what the standard's own helper means:

```daml
accountParties : Party -> HoldingV2.Account -> [Party]
accountParties admin account = case account.provider of
    Some provider | provider /= principal -> [principal, provider]
    _ -> [principal]
  where principal = fromOptional admin account.owner
```

The principal (the owner, or the admin for a special account with `owner = None`), plus the provider only where one is set and differs. For a basic account (`basicAccount`, no provider) it is one party. Nothing here may assume a provider exists.

Source: `token-standard/splice-token-standard-utils/daml/Splice/TokenStandard/Utils/Internal/Conversions.daml:147-170`.

## 3. What the standard gives us, and where this attaches

| We reuse | From | Why it matters here |
| --- | --- | --- |
| `Account { owner : Optional Party, provider : Optional Party, id : Text }` | `HoldingV2.daml` | an entitlement is owed *to an account*, so the custody relationship is explicit |
| `InstrumentId { admin, id }` | `HoldingV2.daml` | the admin is the only party that may announce an event |
| `HoldingView { account, instrumentId, amount, lock, meta }` | `HoldingV2.daml` | the derivation input |
| `accountParties` | `TokenStandard/Utils` | the stakeholder rule we adopt for entitlements |
| `cip-112/mint`, `cip-112/burn` accounts | CIP-0112 §4.3.2.1 | redemption is an allocation leg to the burn account; `BurnMintFactory` is not used |
| `TransferFactory_Transfer`, `Allocation` | `TransferInstructionV2.daml`, `AllocationV2.daml` | the movements `Entitlement_Settle` performs (§6) |
| `ExtraArgs { context : ChoiceContext, meta : Metadata }` | `MetadataV1.daml` | registry-provided context fetched over the off-ledger API |
| Factory + `_PublicFetch` + `actors : [Party]` + `availableActions` | `TransferInstructionV2.daml`, `AllocationV2.daml` | the V2 shape wallets already render |
| Metadata keys prefixed with the defining app's DNS name (`MetadataV1.daml:57-61`, SHOULD) | `MetadataV1.daml` | our references follow it |

What is missing:

- `Lock` has four fields: `holders : [Party]`, `expiresAt`, `expiresAfter` and `context : Optional Text`, the last documented as wallet display and possibly visible more widely than the contracts it describes. It cannot say which obligation it secures, or who gets the coupon.
- No event, no record time, no entitlement in CIP-0056 or CIP-0112.

**V1 and V2 conventions differ by design, and we follow V2.** `AllocationV2`'s module header states that V2 "removes the need to specify the `expectedAdmin` party, as its value is already specified in other choice arguments or not relevant (for `_PublicFetch` choices)". V2 packages take `actors : [Party]` and no `expectedAdmin`; V1-generation packages, including `burn-mint-v1`, take `expectedAdmin` and one `actor`. A V2 token carries `interface instance V1.Holding` and `interface instance V2.Holding` on the same template, so one registry serves both generations.

Source: `splice-api-token-allocation-v2/daml/Splice/Api/Token/AllocationV2.daml:18-19`; `examples/splice-test-token-v2/daml/Splice/Testing/Tokens/TestTokenV2/Holding.daml:31-35`.

**`_PublicFetch` belongs on factories only.** All six declarations in 0.8.3 are on factories (`TransferFactory` V1/V2, `AllocationFactory` V1/V2, `SettlementFactory`, `BurnMintFactory`), and `EventLogView`'s doc comment states the rule for data contracts: "`EventLog` contracts do not support public fetching, as they are intended to be used by the instrument admin or app provider only." Our factory carries it; `LifecycleEvent`, `Entitlement` and `EntitlementRedirect` do not. A V2 `_PublicFetch` validates no admin: the caller compares the returned view against its own trusted source of the admin, which V1 had the caller pass in as `expectedAdmin`.

Source: `git grep -n "choice .*_PublicFetch" 0.8.3 -- 'token-standard/splice-api-token-*' 'daml/splice-api-token-burn-mint-v1'`; `TransferEventsV2.daml:109-112`; `BurnMintV1.daml:58-70`.

### 3.1 Daml Finance and the shipped reference applications

A registry that models its bond in Daml Finance already lifecycles it there: a lifecycle rule evaluates the instrument and produces an effect stating, per unit of the target instrument, what is consumed and what is produced, with a settlement time. That effect is internal to the registry's own application. The servicing interfaces are its cross-application projection, and an adapter maps one onto the other:

| Daml Finance, inside the registry | Servicing, across applications |
| --- | --- |
| the effect's target instrument | `EventKey.instrumentId` |
| a cash quantity produced per unit | `ratePerUnit`, `payInInstrument` |
| the effect's settlement time | `paymentTime` |
| the target instrument consumed, nothing else produced but cash | `EK_Redemption` |
| — | `recordTime`: Daml Finance settles per holding against an instrument version and has no record instant, so the adapter fixes one |
| — | entitlements, redirects, `paidAmount`: no counterpart; the adapter derives and records them |

Milestone 1 publishes this mapping against the current Daml Finance release, field by field; Milestone 3 ships the adapter, for the reference registry's coupon and redemption.

Digital Asset's `ex-asset-servicing` reference application, now archived and built on SDK 1.18, is the precedent for these workflows (coupons, redemptions, dividends and splits). On one point it takes the opposite approach, and that needs an answer. There the issuer publishes per-unit lifecycle effects and bumps the instrument's version; each holder, or its bank, then applies the effects to its own position, archiving the old version and creating the new one with the cash beside it. The issuer never learns who holds, and that privacy is the application's stated aim. There is no record instant: entitlement is holding the old version when it is converted, as in Daml Finance after it.

We derive at the registry instead, for three reasons:

- **The holder cannot pay itself.** In the standard's own reference token a holding is signed by the admin together with the account parties (§13), so a holder cannot create its own cash; the coupon has to arrive as a transfer from whoever holds the cash, which is the paying agent.
- **The privacy it buys is already spent.** The instrument admin of a token-standard instrument already sees every holding of it. Self-processing would hide nothing from the registry that the registry does not already see.
- **It pays the wrong party under a pledge.** A holder processing its own locked holding receives the coupon itself, and the pledgee depends on the holder forwarding it. Paying the pledgee directly needs someone who sees both the position and the redirect, and only the registry does.

Its own `Entitlement` template is a different thing from ours: an instrument conveying the right to receive an asset at a settlement date, not a record of who is owed how much as of a record time. All of it also lives inside one application, with templates no second application is meant to read. This proposal addresses that gap.

## 4. The objects

Three objects, five interfaces: `LifecycleEvent`; `Entitlement`; `EntitlementRedirect` with `RedirectInstruction` and `RedirectFactory`.

Factory naming follows the shipped convention (`TransferFactory` / `TransferFactory_Transfer` / `TransferFactoryView`), so the redirect factory is `RedirectFactory` / `RedirectFactory_Propose` / `RedirectFactoryView`.

### 4.0 Identity: business keys, not contract ids

Canton 3.x has no contract keys, and every object here changes state at least once. Changing a field archives the contract and creates a new one, so a stored `ContractId` to one of these objects dangles as soon as the correction flow runs.

```daml
data EventKey = EventKey with
    admin        : Party
    instrumentId : InstrumentId
    eventId      : Text
  deriving (Eq, Ord, Show, Serializable)

data RedirectKey = RedirectKey with
    admin         : Party
    instrumentId  : InstrumentId
    holderAccount : Account
    redirectId    : Text
  deriving (Eq, Ord, Show, Serializable)

data EntitlementKey = EntitlementKey with
    event    : EventKey
    account  : Account               -- ^ the account that held at record time
    redirect : Optional RedirectKey  -- ^ None = the holder's own share; Some r =
                                     --   the slice allocated under redirect r (4.3)
  deriving (Eq, Ord, Show, Serializable)
```

An entitlement is identified by the slice it pays, not by where it pays. Two redirects from one account to the same beneficiary account produce two entitlements with different keys, each naming its redirect; the beneficiary account is a field of the view (`payableTo`), and two slices may share it.

References between objects are keys. A `ContractId` may accompany a key as a resolution hint, never as the reference itself.

**The key is authoritative.** Where a view carries both a key and a field the key already contains, the key wins; implementations MUST populate them consistently and the conformance suite asserts it.

**Registry obligations.** These are not ledger invariants: the suite can observe a registry violating one, but nothing prevents it.

- `eventId` is unique within `(admin, instrumentId)`; `redirectId` within a `RedirectKey`'s other fields.
- `ref` (§4.2) is unique within an event.
- All entitlements of one event carry `terms` equal under the derived `Eq`. No single party can check this, because the event's observers are narrow and a holder sees only its own copy; the suite sees every party and asserts it across them.

### 4.1 Lifecycle event

No event factory: only the admin may announce, so there is no non-admin initiator for a factory to serve.

```daml
data EventKind
  = EK_CashDistribution   -- coupon, dividend, interest
  | EK_Redemption         -- full redemption at maturity or on a call
  deriving (Eq, Ord, Show, Serializable)

data EventStatus
  = ES_Announced
  | ES_Confirmed      -- ^ derivation complete: every entitlement of the event exists
  | ES_Corrected with supersededBy : EventKey
  | ES_Cancelled with replacedBy : Optional EventKey  -- ^ set when the cancellation
                                                      --   replaces the event (7)
  deriving (Eq, Show, Serializable)

data RoundingDirection
  = RD_HalfEven | RD_HalfUp | RD_Down | RD_Floor | RD_Ceiling
  deriving (Eq, Ord, Show, Serializable)

data RoundingRule = RoundingRule with
    decimals  : Int                -- ^ scale of the payment instrument, 0..10 (5.2)
    direction : RoundingDirection
  deriving (Eq, Show, Serializable)

data EventTerms = EventTerms with
    kind            : EventKind
    recordTime      : Time            -- ^ 5.1
    paymentTime     : Time
    ratePerUnit     : Decimal         -- ^ coupon or redemption price per unit
    payInInstrument : InstrumentId
    rounding        : RoundingRule
  deriving (Eq, Show, Serializable)

data LifecycleEventView = LifecycleEventView with
    key            : EventKey
    terms          : EventTerms
    announcedAt    : Time
    synchronizerId : Text            -- ^ the synchronizer recordTime refers to (5.1)
    status         : EventStatus
    supersedes     : Optional EventKey
    meta           : Metadata
  deriving (Eq, Show, Serializable)

interface LifecycleEvent where
  viewtype LifecycleEventView
```

`RoundingDirection` is our own enum because `DA.Numeric.RoundingMode` has no `deriving` clause, so it is not serializable and cannot appear in a view. The standard fixes five of the stdlib's eight modes, mapping onto `RoundingHalfEven`, `RoundingHalfUp`, `RoundingDown` (toward zero), `RoundingFloor` and `RoundingCeiling`. `RoundingUp` and `RoundingHalfDown` are not offered: no settlement convention we found uses them, and each extra mode is another way two implementations can differ. `RoundingUnnecessary` is excluded for a different reason: it is not a rounding convention. It asserts that no rounding is needed and raises a runtime error when that is false.

**Stakeholders.** Signatory: `key.admin`. Observers are implementation-defined and narrow: the admin's own operating parties and any party the registry chooses to disclose to. Normatively, the holder set is not among them.

A contract's stakeholders are visible to its stakeholders, so an event observed by every holder's account parties would tell every holder and every provider who else holds that instrument. That amounts to the complete holder register, a larger disclosure than anything in §9. The rule would also be unsatisfiable at announcement, because the entitled set is only fixed at `recordTime`: a party buying between announcement and record time would not be an observer of the event that entitles it.

An event therefore reaches its audience in two ways, neither through a wide observer set:

- **Before the record time**, as data over the off-ledger announcement endpoint (§8), for display. A wallet showing "XS123 pays on 31 Dec" needs only the terms.
- **After derivation**, as `EventTerms` denormalized onto each holder's own entitlement, which discloses that holder's line and nothing about any other holder.

**Which copy is authoritative.** The copy on the entitlement: it is signed by the admin, it is on the holder's own participant, and the payment is checked against it. Version 1 has no elections, so a holder acts on nothing it reads from the announcement endpoint, and an endpoint that served wrong terms could mislead a display but not a payment. A correction is a new event with its own `EventKey` linked by `supersedes` / `supersededBy` (§7). The event key therefore identifies the version, and an entitlement is bound to a single version of the terms, by its key and by its copy.

A registry whose holder set is already public MAY widen the observers; that is its choice and its disclosure.

**`recordTime` is an instant, not a date.** A record *date* is a calendar convention over a timezone; the ledger has neither. The registry publishes the date in its documentation and the instant on the contract, and the suite asserts they correspond. §5.1 defines what the instant means operationally.

**Status.** `ES_Announced` from announcement until derivation; `ES_Confirmed` once every entitlement of the event exists; after that the terms change only by a correction (§7). `ES_Corrected` and `ES_Cancelled` are §7's.

**Clocks.** `announcedAt` and `paymentTime` are ledger times as recorded by the registry, and are informational. `recordTime` alone is normative and is defined against record time at the synchronizer (§5.1).

### 4.2 Entitlement

One per `EntitlementKey`.

```daml
data DeductionKind = DK_Withholding | DK_Fee | DK_Custom with id : Text
  deriving (Eq, Ord, Show, Serializable)

data DeductionBasis
  = DB_Gross       -- ^ the entitlement's grossAmount
  | DB_Remaining   -- ^ gross less the deductions earlier in the list
  | DB_Declared    -- ^ an amount the registry declares; `source` says why
  deriving (Eq, Ord, Show, Serializable)

data Deduction = Deduction with
    kind      : DeductionKind
    basisKind : DeductionBasis
    basis     : Decimal      -- ^ the amount the rate was applied to
    rate      : Decimal
    amount    : Decimal      -- ^ round(basis * rate) per the event's rule
    source    : Text         -- ^ a credential, a standing instruction, a default.
                             --   Declared, not derived. See 9.
  deriving (Eq, Show, Serializable)

data EntitlementStatus
  = ETS_Live
  | ETS_Superseded with supersededBy : EntitlementKey
  | ETS_Cancelled with replacedBy : Optional EntitlementKey
  deriving (Eq, Show, Serializable)

data EntitlementView = EntitlementView with
    key           : EntitlementKey
    ref           : Text         -- ^ opaque, registry-assigned, unique within the event
    payableTo     : Account      -- ^ key.account, or the redirect's beneficiary
    terms         : EventTerms   -- ^ denormalized from the event; 4.1
    payingAgent   : Party
    unitsAtRecord : Decimal      -- ^ units attributed to THIS entitlement; 5.1
    grossAmount   : Decimal      -- ^ round(units * rate); 5.2
    deductions    : [Deduction]
    netAmount     : Decimal      -- ^ = grossAmount - sum(deductions.amount)
    paidAmount    : Decimal      -- ^ sum settled through Entitlement_Settle,
                                 --   carried across supersession (6, 7)
    derivedFrom   : [ContractId Holding]  -- ^ hint only
    supersedes    : Optional EntitlementKey
    status        : EntitlementStatus
    meta          : Metadata
  deriving (Eq, Show, Serializable)

data EntitlementSettleResult = EntitlementSettleResult with
    paidAmount : Decimal   -- ^ the successor's paidAmount
    meta       : Metadata
  deriving (Eq, Show, Serializable)

interface Entitlement where
  viewtype EntitlementView
  entitlement_settleImpl
    : ContractId Entitlement -> Entitlement_Settle -> Update EntitlementSettleResult
  entitlement_settleExtraObservers : Entitlement_Settle -> [Party]

  -- | Pays `amount` towards this entitlement and records it, in one transaction.
  -- Implementations MUST: require `actors == [payingAgent]`, `status == ETS_Live`,
  -- the event live (`ES_Announced` or `ES_Confirmed`, fetched from the context)
  -- and `0 < amount <= netAmount - paidAmount`, with equality for `EK_Redemption`
  -- (no partial redemption payment); move `amount` of `payInInstrument`
  -- to `payableTo` inside this choice, by a movement that completes in this
  -- transaction; archive this contract and create a successor with `paidAmount`
  -- increased by `amount`. Either both happen or neither does (6).
  nonconsuming choice Entitlement_Settle : EntitlementSettleResult
    with
      actors    : [Party]
      amount    : Decimal
      extraArgs : ExtraArgs   -- ^ the servicing registry's context, with the cash
                              --   registry's transfer-factory context nested (6)
    observer entitlement_settleExtraObservers this arg
    controller actors
    do entitlement_settleImpl this self arg
```

**What is owed, at any moment,** is `outstanding = netAmount - paidAmount`, which is computed and never stored. Positive, the entitlement is open for that much; zero, settled; negative, overpaid, and the difference is a claim of the registry on `payableTo` (§7). Storing a settlement state beside the two amounts would be a third field that could disagree with them.

**Stakeholders, normative.** Signatory: `key.event.admin`. Observers: `accountParties admin key.account` ∪ `accountParties admin payableTo` ∪ `[payingAgent]`. The paying agent is included because it settles the entitlement and otherwise could not read it.

**Storage cost.** One persistent contract per entitlement on every stakeholder's participant, re-created once per payment and once per correction. An instrument with 50,000 holders paying quarterly derives 200,000 entitlements a year and creates at least 400,000 contracts for them (the derivation and at least one payment each), each replicated to the holder's and any provider's participant. Settled entitlements stay active as the record of what was paid, so the live set grows by about 200,000 a year until a registry archives closed events under its own retention policy.

Eager register creation (§4.3) adds one more per `(account, instrument)`: 50,000 on that instrument, observed by the holder's account parties, whether or not a redirect is ever proposed. Against 200,000 entitlements a year that is a quarter as many contracts, not an order of magnitude fewer, though unlike the entitlements it is a one-off cost and not a per-event one. It is the price of serializing the first completed redirect on a position as well as later ones. §12.3 puts the entitlement alternative to the SIG; the register has no alternative to put.

This rule is ours. `EventLog_HoldingsChange`'s "observers MUST include at least the account parties" constrains the observers of a nonconsuming choice, which is a transient disclosure on a transaction, and says nothing about the stakeholders of a persistent contract.

Source: `TransferEventsV2.daml:92-95`.

**One state-changing choice, and no `_PublicFetch`.** An entitlement is a statement of what is owed and what has been paid. `Entitlement_Settle` is the only way the second figure changes; supersession and cancellation are the registry's own operations on its own contract (§7). Its readers are its observers, who read it off their own participant; a party that is not an observer has no disclosure source for it.

**`derivedFrom` is a hint.** By payment time those holdings are archived and cannot be fetched. Reproducing the figure needs the ACS at the record offset or the update history. `derivedFrom` lets a party that has the history see which contracts the registry used; it does not let a party without the history verify anything. §5.5 states what verification requires.

### 4.3 Entitlement redirect

```daml
data RedirectStatus
  = RS_Live
  | RS_Revoked with revokedAt : Time
  | RS_Expired
  deriving (Eq, Show, Serializable)

data RedirectScope = RedirectScope with
    instrumentId : InstrumentId
    unitsCap     : Optional Decimal  -- ^ None = the whole position; then no other
                                     --   redirect may be live, and toTime MUST be set
    kinds        : [EventKind]       -- ^ [] = all kinds
    fromTime     : Time              -- ^ compared against the event's recordTime
    toTime       : Optional Time     -- ^ likewise
  deriving (Eq, Show, Serializable)

data EntitlementRedirectView = EntitlementRedirectView with
    key          : RedirectKey
    beneficiary  : Account
    scope        : RedirectScope
    priority     : Int               -- ^ handed out by the register at completion
    status       : RedirectStatus
    revokeRequestedBy : [Party]      -- ^ holder-side parties that asked to revoke
    agreementRef : Optional AnyContractId  -- ^ id only; see 9
    meta         : Metadata
  deriving (Eq, Show, Serializable)

interface EntitlementRedirect where
  viewtype EntitlementRedirectView
  entitlementRedirect_revokeImpl
    : ContractId EntitlementRedirect -> EntitlementRedirect_Revoke -> Update RedirectResult
  entitlementRedirect_revokeExtraObservers : EntitlementRedirect_Revoke -> [Party]
  entitlementRedirect_expireImpl
    : ContractId EntitlementRedirect -> EntitlementRedirect_Expire -> Update RedirectResult
  entitlementRedirect_expireExtraObservers : EntitlementRedirect_Expire -> [Party]

  -- | One party per call, from either side. A holder-side call records a request;
  -- any beneficiary-side call completes the revocation (4.3).
  nonconsuming choice EntitlementRedirect_Revoke : RedirectResult
    with
      actors    : [Party]
      extraArgs : ExtraArgs
    observer entitlementRedirect_revokeExtraObservers this arg
    controller actors
    do entitlementRedirect_revokeImpl this self arg

  nonconsuming choice EntitlementRedirect_Expire : RedirectResult
    with
      actors    : [Party]
      extraArgs : ExtraArgs
    observer entitlementRedirect_expireExtraObservers this arg
    controller actors
    do entitlementRedirect_expireImpl this self arg

-- | What a pending instruction proposes; the redirect's priority and status
-- exist only once it completes.
data RedirectProposal = RedirectProposal with
    key          : RedirectKey
    beneficiary  : Account
    scope        : RedirectScope
    agreementRef : Optional AnyContractId
    meta         : Metadata
  deriving (Eq, Show, Serializable)

data RedirectInstructionView = RedirectInstructionView with
    admin            : Party
    proposal         : RedirectProposal
    approvedBy       : [Party]       -- ^ approvals collected so far, proposer included
    availableActions : Map.Map RedirectAction [[Party]]
                                     -- ^ every group under RA_Approve is one party
    expiresAt        : Optional Time
    meta             : Metadata
  deriving (Eq, Show, Serializable)

interface RedirectInstruction where
  viewtype RedirectInstructionView
  redirectInstruction_approveImpl
    : ContractId RedirectInstruction -> RedirectInstruction_Approve -> Update RedirectResult
  redirectInstruction_approveExtraObservers : RedirectInstruction_Approve -> [Party]

  -- | Records the approval of `actors`, which MUST be one of the single-party
  -- groups listed under `RA_Approve`. Returns Pending with the successor instruction while approvals
  -- are missing; the approval that completes the set creates the redirect.
  nonconsuming choice RedirectInstruction_Approve : RedirectResult
    with
      actors    : [Party]
      extraArgs : ExtraArgs
    observer redirectInstruction_approveExtraObservers this arg
    controller actors
    do redirectInstruction_approveImpl this self arg

  -- _Reject and _Withdraw are identical in shape — their methods, observer
  -- methods and choices are elided here — callable by any beneficiary-side and
  -- any holder-side account party respectively.

data RedirectAction = RA_Approve | RA_Reject | RA_Withdraw
  deriving (Eq, Ord, Show, Serializable)

data RedirectResult_Output
  = RedirectResult_Pending with instructionCid : ContractId RedirectInstruction
  | RedirectResult_Completed with redirect : RedirectKey
  | RedirectResult_Failed
  deriving (Eq, Show, Serializable)

data RedirectResult = RedirectResult with
    output : RedirectResult_Output
    meta   : Metadata
  deriving (Eq, Show, Serializable)

interface RedirectFactory where
  viewtype RedirectFactoryView
  redirectFactory_proposeImpl
    : ContractId RedirectFactory -> RedirectFactory_Propose -> Update RedirectResult
  redirectFactory_publicFetchImpl
    : ContractId RedirectFactory -> RedirectFactory_PublicFetch -> Update RedirectFactoryView
  redirectFactory_proposeExtraObservers : RedirectFactory_Propose -> [Party]

  nonconsuming choice RedirectFactory_Propose : RedirectResult
    with
      holderAccount : Account
      beneficiary   : Account
      scope         : RedirectScope
      agreementRef  : Optional AnyContractId
      actors        : [Party]
      extraArgs     : ExtraArgs
    observer redirectFactory_proposeExtraObservers this arg
    controller actors
    do redirectFactory_proposeImpl this self arg

  nonconsuming choice RedirectFactory_PublicFetch : RedirectFactoryView
    with actors : [Party]
    controller actors
    do redirectFactory_publicFetchImpl this self arg

data RedirectFactoryView = RedirectFactoryView with
    admin : Party
    meta  : Metadata
  deriving (Eq, Show, Serializable)
```

Every state-changing choice here is nonconsuming and returns a result, as are all fifteen state-changing choices in the shipped V2 interfaces; the fourteen outside `EventLog` also carry an `…ExtraObservers` method. `_Revoke` and `_Expire` MUST archive the redirect and create a successor carrying `RS_Revoked` or `RS_Expired` (or, for a recorded revocation request, the same status with `revokeRequestedBy` extended). A consumed contract has no view, so a revocation that only archived would be indistinguishable from absence. `_Expire`'s `actors` MUST be the admin, and the implementation MUST reject unless `scope.toTime` is set and has passed; without that check the admin could archive any live redirect at will. A redirect with `toTime = None` is never expirable.

**What a redirect is.** An instruction, approved by the holder's side and the beneficiary's side, that income derived from the units held in `holderAccount` at an event's record time is paid to `beneficiary`, up to `unitsCap`. Its only source is the account's position: if the account holds nothing at the record time, the redirect yields nothing. A pledge does not by itself decide who receives income. The pledge agreement says what the parties agreed and the lock says the units cannot move; only a completed redirect tells the registry to pay someone other than the holder. An obligation to pay a counterparty an amount regardless of what the account holds (the equivalent payment a securities borrower owes a lender after selling the borrowed bond) is not a redirect, and version 1 does not model it (§13).

**Authorization: one party per command.** The holder's account parties, the beneficiary's account parties and the registry are, in the case this standard exists for, on different participants, and a command can only act as parties its participant hosts. No choice here therefore needs two account parties in one command. The flow follows the shipped transfer instruction, where `TransferFactory_Transfer` "SHOULD … allow all the sender's account parties to call this choice, so they can all initiate a transfer on their own", and further approvals arrive as steps on a pending instruction:

- **Propose.** Any one holder-side account party exercises `RedirectFactory_Propose` alone. The holder side is `accountParties admin holderAccount`: the principal and, where one is set and differs, the provider. The registry's authority comes from its own factory, so the registry is never a separate step. The result is a pending `RedirectInstruction` whose `approvedBy` holds the proposer.
- **Approve.** Every further approval is one `RedirectInstruction_Approve` by one party, from its own participant. `availableActions` lists, under `RA_Approve`, each party whose approval is still missing, one party per group. Which further account parties must approve is the registry's account policy, as it is for transfers; the standard fixes the minimum: the principal of the holder account and the principal of the beneficiary account both approve. A provider therefore cannot redirect its client's income on its own authority, even to an account it owns itself as in the Milestone 1 fallback (§9), where the provider is the whole beneficiary side but the holder still approves. Each approval archives the instruction and creates a successor with `approvedBy` extended, until the approval that completes the set creates the redirect.
- **Reject and withdraw.** Any beneficiary-side party may reject and any holder-side party may withdraw, alone, archiving the instruction.
- **Revoke.** `EntitlementRedirect_Revoke` is likewise called by one party. A holder-side call records a request in `revokeRequestedBy` and changes nothing else; a beneficiary-side call (the side the redirect benefits, releasing its own claim) completes the revocation, whether or not a request was recorded. A holder can therefore never revoke alone, a beneficiary always can, and neither needs the other in its command.

CIP-0112 mandates provider visibility on asset movements and holdings, and a redirect is neither; whether a provider's approval is required is therefore left to the registry's policy. On a basic account there is no provider and the principal alone represents the holder's side.

**Stakeholders.** `EntitlementRedirect`: signatory `key.admin`, the same party that signs the register, which makes the exclusivity check enforceable in one transaction; observers `accountParties admin key.holderAccount` ∪ `accountParties admin beneficiary`. `RedirectInstruction`: signatory `admin`; observers the same two sets, so every party that may approve can see what it approves from its own participant. §9's rows for the registry, the two providers and the beneficiary are claims about these sets. The holder's provider is therefore an observer of every redirect on its client's position, which is why it is the counterparty that can recompute the whole split (§5.5).

**Scope is by account and instrument with an optional cap, never by holding id.** Holdings are archived and re-created on every transfer, split, merge, lock and unlock, so a redirect pinned to a `ContractId Holding` stops matching the moment the position is touched, including by the lock the pledge itself creates.

**Which events a redirect catches** is decided by the snapshot as well as the scope (§5.1). The redirect must be live in the ACS at the record offset, and its scope must then match: the event's kind is in `kinds` (or `kinds` is empty), `fromTime <= recordTime`, and `toTime` is `None` or `recordTime <= toTime`. So `fromTime` can narrow a redirect but never extend it backwards: a redirect completed after a record time does not catch that event, whatever `fromTime` it carries.

**Allocation under competing redirects.** Priority is the `Int` the register hands out when a redirect completes, which is monotone and therefore a total order with no tie-break needed. It is fixed then and never re-evaluated, so a redirect completed later can never take units from one already live. `fromTime` plays no part: it is a scope filter, holder-supplied, so ranking on it would let a proposer jump the queue.

Derivation allocates in priority order among the redirects that catch the event: each takes up to its cap until the position is exhausted, and the holder keeps the remainder. If the position is below the committed caps, the holder's own share absorbs the shortfall first, down to zero, and only then does the lowest-priority redirect receive less than its cap. Each entitlement names the redirect it was allocated under in its key.

**Uncapped redirects.** A redirect with `unitsCap = None` claims the whole position. It may complete only if nothing else is live on that position, it blocks every later completion while it lives, and it MUST carry a `toTime`; without one, a beneficiary that never revokes would encumber the position permanently.

**Capped redirects** may coexist without limit, and their caps are not checked against the position, for the reason below. One with no `toTime` is never expirable, so its slice can be encumbered the same way; the exposure is the cap in units. We accept that and do not force a deadline on every redirect, because a pledge of a fixed number of units with no stated end is an ordinary arrangement.

**Exclusivity is per position, whatever the scopes.** An uncapped redirect over coupons only still blocks a capped one over redemptions only, and one whose window has not started still blocks one whose window has ended. The reason is that the check must be answerable from the register, which carries aggregates and no scopes, and a register that carried scopes would disclose them to every approving beneficiary (§9). Parties that want disjoint uncapped redirects on one position cap them instead.

**Serialization, and what it does and does not prevent.** Daml cannot check non-existence, so "no conflicting redirect exists" is unenforceable against concurrent commands. Completion must therefore be serialized per position, through a contract the registry consumes and re-creates. Only the completing approval touches it; earlier approvals change the instruction alone.

**That contract is not in this package.** An interface package cannot declare a template: damlc rejects it with "This package defines both interfaces and templates. This may make this package and its dependents not upgradeable", and none of the thirteen shipped `splice-api-token-*` packages contains a `template`. The standard therefore specifies the obligation and the aggregate the completing approval's choice context must supply, and the register itself is a registry-side template. The reference registry's version:

```daml
template RedirectRegister with
    admin          : Party
    holderAccount  : Account
    instrumentId   : InstrumentId
    committedUnits : Decimal   -- ^ sum of live caps
    liveCount      : Int
    uncappedLive   : Bool      -- ^ an aggregate cannot otherwise distinguish a
                               --   live uncapped redirect from a live zero-cap one
    nextPriority   : Int       -- ^ handed out and incremented at completion
  where signatory admin
        observer accountParties admin holderAccount
```

Normative for any registry, whatever it calls its own register:

- Exactly one live register per `(holderAccount, instrumentId)`, signed by the admin and observed by the holder's account parties, so a holder can always see its own committed total.
- **First created only by commands the registry itself submits**, never inside another party's transaction, such as the receiving leg of a transfer, a settlement or a mint, whose commands carry someone else's submitter and command id. The registry creates it eagerly: at the account's setup, for an instrument with explicit accounts; by its own automation on observing the first holding in the account, for one with implicit accounts (CIP-0112 §4.3.2.2: "Some assets like Canton Coin will use implicit accounts only"); and in any case before it serves a choice context for the account, including a `_Propose` context, which covers arranging a pledge before the position arrives.
- **Creation must not race**, and the ledger will not stop it: two creates of the same register are two valid commands. Because every creation is the registry's own command, the registry can serialize them: it submits them all as one user from one participant, with a command id derived deterministically from `(admin, holderAccount, instrumentId)`, so the Ledger API's command deduplication (scoped to the participant, the submitting user, the acting parties and the command id, all fixed here) rejects the second of two concurrent creations; and before submitting, the registry checks its own active contract set for a live register. Deduplication alone is bounded by its period and the check alone races; together they leave a window only if a register is created, archived without a successor, and created again, and no operation here does that.
- A redirect cannot complete without a register: the completing approval consumes it, so a position with none has no completed redirects.
- Consumed and re-created by the completing `_Approve`, a completing `_Revoke` and `_Expire`; never by `_Propose`, by a non-completing approval, or by a revocation request.
- The register is disclosed to an approving beneficiary-side party, and a disclosed contract carries its full create argument (`DisclosedContract.createdEventBlob` is "the serialized created event"). The beneficiary therefore learns the whole payload, `nextPriority` included, and no arrangement of fields changes that: an admin-only side contract would have to be disclosed too for the beneficiary's participant to exercise over it, and the beneficiary is a witness of its archive regardless. §9 states the disclosure.

How each choice moves the aggregates, so that any two registries agree on them:

| Choice | `committedUnits` | `liveCount` | `uncappedLive` | `nextPriority` |
| --- | --- | --- | --- | --- |
| completing `_Approve`, capped at *c* | + *c* | + 1 | unchanged | handed to the redirect, then + 1 |
| completing `_Approve`, uncapped | unchanged | + 1 | `True` | handed to the redirect, then + 1 |
| completing `_Revoke` / `_Expire`, capped at *c* | − *c* | − 1 | unchanged | unchanged |
| completing `_Revoke` / `_Expire`, uncapped | unchanged | − 1 | `False` | unchanged |

Two proposals on one position both succeed, and so do any number of non-completing approvals; two *completions* race, and one loses.

Expiry needs `_Expire` because `toTime` passing runs no transaction: nothing would decrement `committedUnits`, and the capacity would stay committed. Until the registry sweeps, an expired redirect still occupies its place in the register, but not in any entitlement, because the scope rule above already excludes events after `toTime`.

The two outcomes are different and the suite tests both:

- **Contention.** The losing completion fails because a contract it used is no longer active. This is reported as contract-not-active when the command is interpreted, or as an inactive or locked contract when it is confirmed; the suite keys on that category, not on one error code. The losing party re-fetches the choice context and retries. The retry succeeds unless the business rule now rejects it.
- **Business rejection.** An uncapped redirect is live, or this redirect is itself uncapped and something else is live. Retrying will not help, and a wallet shows why. A retry can also end this way: when either of two racing redirects is uncapped, the loser's retry is rejected.

The completing approval is a choice on `RedirectInstruction`, so the register reaches that party as a disclosed contract in that instruction's choice context (§8, `…/redirect-instructions/{id}/choice-contexts/approve`).

**There is no cap-against-position check, at completion or anywhere.** There is no place to put one: checking it on-ledger needs the position in the transaction, which means disclosing the holder's holdings to an approving beneficiary (a disclosure the design otherwise avoids, §9), and moving it off-ledger makes it a rule a Daml conformance suite cannot observe.

It is also unnecessary. The allocation rule above already defines what happens when committed caps exceed the position: allocation in priority order, the holder's share absorbing the shortfall first. Over-commitment at completion is the same situation as a position that falls afterwards, and the second cannot be prevented by any check. The caps are therefore permitted to exceed the position, and `committedUnits` is information, not a limit.

`committedUnits` tells a beneficiary how much is committed ahead of it, since every redirect live at its completion has a lower priority number. It does not tell it whether the position covers that amount plus its own cap, because the beneficiary never sees the position. Coverage is the pledge agreement's business, checked by whatever means that agreement provides; the register does not establish it.

What the choice body does check on-ledger, and the suite therefore tests, is the uncapped-exclusivity rule: an uncapped redirect may complete only when `liveCount = 0`, and none may complete while `uncappedLive`. Both conditions are answerable from the register alone. Concurrency can break this invariant, and the serialization exists to protect it.

The register carries aggregates only, never the list of redirects, their beneficiaries or their scopes.

Contention is bounded to one holder's position in one instrument.

## 5. Derivation

### 5.1 The snapshot

**What is read.** The registry establishes two sets from the ACS at the ledger offset corresponding to `recordTime` (or by folding the update stream to that offset): the holdings of the instrument, and the redirects on those positions with their status. It does not use an on-ledger `fetch`, which only sees currently active contracts. Both sets come from the same offset, so holdings and redirects are never read at different moments.

**The boundary.** Activeness is decided by record time at the synchronizer, not by the ledger time a submission requested. A contract is in the snapshot iff it was created by a transaction whose record time is at or before `recordTime`, and not archived by one whose record time is at or before `recordTime`. So a holding created at exactly `recordTime` counts, and one archived at exactly `recordTime` does not; one archived after it does, because it was live at the instant.

**What follows for redirects.** A redirect counts iff the contract active at the record offset carries `RS_Live` and its scope matches the event (§4.3). Consequences, each asserted by the suite:

- a redirect completed after `recordTime` does not catch the event, whatever its `fromTime`;
- a redirect revoked or expired after `recordTime` still catches it, because the revocation is the successor contract, which did not exist at the offset;
- a redirect revoked before `recordTime` does not.

**Locked holdings count.** A pledged bond is still held.

**Completeness.** A participant has a complete view through `recordTime` once it has ingested an update whose record time is later than `recordTime` on the event's synchronizer; until then a late transaction could still change the set. The registry MUST NOT derive before that. A counterparty verifying the figures applies the same rule on its own participant.

**Offsets are local.** An offset is an index into one participant's update stream, and two participants number the same transaction differently. "The offset corresponding to `recordTime`" is therefore each participant's own: the last offset on that participant whose record time is at or before `recordTime`. The standard compares contracts and figures across participants, never offsets.

**One synchronizer.** `LifecycleEventView.synchronizerId` names the synchronizer `recordTime` refers to. If the registry observes holdings of the instrument on any other synchronizer at that offset (or a holding unassigned from one synchronizer and not yet assigned to another), it MUST refuse to derive and resolve the position first. Declaring a synchronizer does not make holders elsewhere unentitled, it only hides them. A reassignment between `recordTime` and `paymentTime` does not affect the snapshot, which is already fixed.

**Retention.** Independent verification needs the history at the record offset. The registry, and any provider that verifies, MUST retain its update history from `recordTime` until every entitlement of the event is settled, superseded or cancelled and no correction of it is pending; pruning earlier makes the figures unverifiable. A participant that migrates or is restored from backup MUST resume from a state that still contains that history. The suite cannot test pruning; the integration guide states it as an operating requirement.

**Units per entitlement.** `unitsAtRecord` on an entitlement is the units attributed to that entitlement, not the account's whole position. Under a partial redirect one account yields two or more entitlements (the holder's share and one per redirect slice), and their `unitsAtRecord` sum to the position at `recordTime`. A capped slice tells its beneficiary the slice and not the position, with two exceptions that §9 names: an uncapped redirect's entitlement *is* the whole position, and a beneficiary whose slice falls short of its cap can work the position out. §8 keeps the holder's holdings out of the approval choice context, so neither exception arises from the redirect flow itself.

### 5.2 The arithmetic, pinned

`Decimal` is `Numeric 10`, and `*` on two `Numeric 10` values yields a `Numeric 10`, rounding the exact product half-even at scale 10. A naive `unitsAtRecord * ratePerUnit` is therefore already rounded before any explicit rounding, and two implementations placing that intermediate differently disagree.

`DA.Numeric.mul` takes two scales and produces a third, so the exact product of two `Decimal`s is available at scale 20 with no rounding at all. Normatively, the product is computed there and rounded once:

```daml
toMode : RoundingDirection -> DN.RoundingMode
toMode = \case
  RD_HalfEven -> DN.RoundingHalfEven
  RD_HalfUp   -> DN.RoundingHalfUp
  RD_Down     -> DN.RoundingDown
  RD_Floor    -> DN.RoundingFloor
  RD_Ceiling  -> DN.RoundingCeiling

grossAmountOf : RoundingRule -> Decimal -> Decimal -> Decimal
grossAmountOf rule units rate =
  DN.cast (DN.roundNumeric rule.decimals (toMode rule.direction) (DN.mul units rate : Numeric 20))
```

One rounding, at the stated scale, in any of the five directions the standard fixes, and it compiles at `--target=2.1`.

Two limits, both normative:

- `rounding.decimals` is in `0..10`; outside that range the implementation rejects the event and the suite asserts the rejection.
- `Numeric 20` represents values strictly between −10^18 and 10^18, so a product at or beyond 10^18 overflows and the implementation MUST reject rather than round. The suite asserts the rejection at that boundary.

**Not `BigNumeric`.** `DA.BigNumeric` exports nothing below LF 2.dev (its module body is behind `#ifndef DAML_BIGNUMERIC`), and every token-standard package is `--target=2.1`. A package built the way the standard is built cannot use it, and a registry shipping a 2.dev DAR would need alpha-version support on every participant and synchronizer that vets it. Verified by compiling at 2.1, 2.2, 2.3 and 2.dev.


Source: `DA/Numeric.daml:28-31` (`mul`), `:216-226` (`roundNumeric`), `:41-42` (`cast`); `DA/BigNumeric.daml:4-11`; `DA/Internal/Prelude.daml:573-591`; token-standard `daml.yaml` files (`--target=2.1`, `sdk-version: 3.5.2`).

### 5.3 Deductions and net

Deductions apply in list order. Each states its basis structurally as `DB_Gross` (the entitlement's `grossAmount`), `DB_Remaining` (gross less the deductions before it in the list) or `DB_Declared` (an amount the registry supplies, with `source` saying why). It records the resulting `basis` value, the `rate`, and `amount = round(basis * rate)` at the event's scale and direction, by the §5.2 expression. A withholding on gross and a fee on the remainder are both expressible, and for the first two kinds a counterparty can recompute `basis` as well as `amount`.

```
netAmount = grossAmount - sum(deductions.amount)
```

**Withholding is an input.** A `Deduction` records where the rate came from: a credential, a standing instruction, a default. It does not compute tax, which is jurisdictional and is the registry's responsibility and liability.

### 5.4 The rounding residual

Each entitlement is rounded on its own, so the sum of entitlements differs from the exact `rate × total units` by a residual of at most one minor unit per entitlement. The residual stays with the registry. It is not allocated to holders, and no entitlement carries a share of it.

The rule keeps every `grossAmount` a pure function of that entitlement's own `unitsAtRecord` and the event terms. A custodian holding only its own clients' positions can recompute each of their lines without knowing anyone else's, and independent recomputation is the acceptance criterion the proposal rests on. Any rule that allocates the residual ranks holders against each other and destroys that property; it belongs in an extension, with the funded-total field it needs (§13).

What this means for a holder who splits a position, with concrete examples. At `decimals = 0` and a rate of 0.4 per unit, one account of 2 units receives `round(0.8)`, and two redirect slices of 1 unit each receive `round(0.4)` twice:

| Direction | One slice of 2 | Two slices of 1 | Effect of splitting |
| --- | --- | --- | --- |
| `RD_HalfUp`, `RD_HalfEven` | 1 | 0 + 0 | −1 |
| `RD_Ceiling` | 1 | 1 + 1 | +1 |
| `RD_Floor`, `RD_Down` | 0 | 0 + 0 | 0 |

So splitting can gain or lose, by at most one minor unit per extra slice, paid by or to the registry and never by or to other holders. A registry choosing `RD_Ceiling` accepts that it pays the gain.

### 5.5 What independent verification is

The counterparty is an observer of the registry's entitlement contract, so comparing that contract with itself proves nothing. What each party derives independently from its own participant, and what the tooling compares:

| Field | Derived from | Holder's provider | Beneficiary |
| --- | --- | --- | --- |
| `unitsAtRecord` | holdings at its own record offset, and the redirects live there | yes: it observes the holdings and every redirect on the position | no: it sees neither the position nor the other redirects |
| `payableTo` | the same | yes | its own slice only |
| `grossAmount` | `unitsAtRecord` + `terms` | yes | yes, given the entitlement's `unitsAtRecord` |
| `deductions` | basis kinds and declared rates | `DB_Gross` and `DB_Remaining`: yes, where it holds the same rate inputs. `DB_Declared`: no, the registry supplies the basis | arithmetic only, given the declared rates |
| `netAmount` | the above | yes | arithmetic only |
| `paidAmount` | the movements `Entitlement_Settle` performed | on the holder's own share, yes, against the receipts into its client's account. On a redirected slice, no: it sees only the registry's successor contract, not the payment into the beneficiary's account | yes, for its own slice, against the receipts into its own account |

So the comparison that evidences the standard is registry against the holder's provider: two independently operated applications, each deriving from its own participant the units, the split, gross, deductions on a gross or remaining basis, net, the payable account, and what was paid on the holder's own share. Declared deduction bases are reported by the registry, not derived. What was paid on a redirected slice is checked by the beneficiary, against its receipts. The beneficiary checks its line's arithmetic and its receipts, not its allocation.

### 5.6 Test vectors

Ship at Milestone 2, before any implementation is final: rounding boundaries in each of the five directions, at scales 0 and 10; the overflow boundary at 10^18; multiple deductions on each of the three basis kinds; zero-entitlement holders; holdings and redirects created and archived on each side of `recordTime`, including at the record time itself; a position split across two accounts with the same provider; a partial redirect; two redirects from one account to the same beneficiary account; a position below the committed caps; and the §5.4 splitting examples.

## 6. How the value moves

Value moves over the existing rails, and every payment is recorded in the transaction that makes it.

**`Entitlement_Settle` pays and records in one transaction.** The paying agent exercises it with an amount. Inside the choice, the implementation moves that amount over the existing rails and archives the entitlement for a successor whose `paidAmount` is higher by that amount. Either both commit or neither does.

| Case | Movement inside `Entitlement_Settle` | Reference carried in the movement's `meta` |
| --- | --- | --- |
| Cash distribution | `TransferFactory_Transfer`, paying agent → `payableTo` | `…/instrument` = the serviced instrument, `…/event` = `eventId`, `…/entitlement` = the entitlement's `ref` |
| Redemption | the cash as for a distribution, in one settlement of the whole outstanding amount; the units retired to the `cip-112/burn` account by CIP-0112's delivery-versus-burn, as the shipped `TestDeliveryVersusBurnMint` performs it, inside the same choice where the holder's allocation exists (see below) | same, both legs |

**What this buys.**

- **No double payment on retry.** A worker that crashes after the transaction commits, and retries, finds the entitlement archived: the retry fails contract-not-active, and the successor's `paidAmount` already includes the payment. A worker that crashes before commit retries a transaction that never happened.
- **Concurrent workers** paying one entitlement both consume the same contract; one commits, the other fails and re-reads.
- **Partial payment** of a distribution is several settlements, each bounded by `netAmount - paidAmount`. An entitlement settled in several transfers records their sum. A redemption is never paid in part: each settlement of an `EK_Redemption` entitlement pays the whole outstanding amount, because paying part of a redemption is partial redemption, which version 1 does not define.
- **A correction and a payment cannot cross.** A correction archives the entitlement too (§7), so the ledger orders the two: either the payment lands first and the correction carries its `paidAmount` forward, or the correction lands first and the payment fails against an archived contract.

**The movement must complete inside the choice.** A transfer factory may return `Pending` when the receiver has to accept, and a pending transfer has moved nothing yet and may never move anything. Recording it as paid would be false, and recording it as in flight would need a later transaction to find out how it ended, which Daml cannot observe for an archived instruction. So `Entitlement_Settle` MUST fail when the movement does not complete in the transaction, and nothing moves. A beneficiary account must therefore be able to receive directly, by whatever arrangement the payment instrument offers for that; §12.5 asks the SIG whether the standard should name one.

**Redemption, step by step.** Retiring units needs the holder's authority (in the standard's reference token a holding is `signatory accountParties admin account, admin`), which neither the paying agent nor the admin has inside `Entitlement_Settle`. The units and the cash are therefore two obligations, specified separately and joined where they can be:

- **The instrument is frozen before `recordTime`.** A choice body reads ledger time, while the snapshot is cut at record time (§5.1), and Canton lets the two differ by up to the synchronizer's ledger-time/record-time tolerance (60 seconds by default). The freeze therefore starts at `recordTime` minus that tolerance, taking the largest tolerance in force between the freeze start and `recordTime`, since it is a dynamic synchronizer parameter. From then, by ledger time, every choice of the registry's own templates that moves the instrument MUST reject the movement unless it goes to `cip-112/burn`. This applies to every such choice and not only the factories, including accepting a transfer instruction created earlier and settling or executing an allocation that already exists. A transaction recorded after `recordTime` has a ledger time after the freeze starts, so none moves the instrument across the record time. The account that held at the record time therefore still holds at payment, and no buyer after the record time acquires units that are about to be retired. The freeze is the token-level form of the end of trading before maturity; the cost is that movements in the last tolerance window before `recordTime` are rejected too.
- **The holder side allocates the units.** Any one holder-side account party allocates the frozen holding to `cip-112/burn`, with the paying agent as executor, as `TestDeliveryVersusBurnMint` has the holder do. A pledged holding is first released by its lock holders; that release and the allocation may be separate transactions, because the freeze leaves the released units nowhere to go but the burn account.
- **The cash is settled in full,** by `Entitlement_Settle`, to `payableTo`: the pledgee if the redirect's `kinds` include `EK_Redemption` or are empty, otherwise the holder.
- **Joined when possible.** Where the holder's allocation exists at settlement, `Entitlement_Settle` executes the burn leg in the same transaction as the cash (delivery versus burn). Where it does not, the cash is paid alone and the registry executes the burn leg when the allocation arrives. An account with several slices has several redemption entitlements but one allocation: whichever settlement first finds the allocation executes the burn leg and consumes it, so the others find it gone and pay cash only. The order does not affect what anyone is owed, because the frozen units can only be burned.

**The cash comes from another registry.** The payment instrument usually has a different admin from the serviced one, so the transfer inside `Entitlement_Settle` runs on the cash registry's factory, with that registry's choice context and disclosed contracts. The paying agent fetches them from the cash registry's own `POST /registry/transfer-instruction/v2/transfer-factory`, as any sender does, and passes them in `extraArgs.context` under a key the servicing package defines; the servicing registry's own entries sit beside them. The command carries the disclosed contracts of both registries.

**The serviced instrument has to be in `meta`.** `eventId` is unique only within `(admin, instrumentId)`, and the transfer's own `instrumentId` is the *cash* instrument while its sender may be a delegated paying agent instead of the admin. Without the serviced instrument in `meta`, a reconciler holding the transfer has neither component of the pair. The three keys carry one value each.

**The reference is short strings, not a key.** `Metadata` is `TextMap Text`, and an `EntitlementKey` is a nested record holding an `Account` with `Optional Party` fields; encoding it would need a serialization this document would have to define, and would put party identifiers into a string every stakeholder of the transfer can read. The entitlement's `ref` (§4.2) is opaque and resolves against the registry, so neither problem arises.

The key prefix follows the namespace decision in §12.1 and is the DNS name of whoever defines the keys; this document does not presume it.

**Contention** on the payment path is on the entitlement and on the holdings the paying agent spends. Both are per payment and not per event, so a payment run over 50,000 entitlements contends only with itself.

## 7. Corrections

Nothing is erased and nothing is mutated in place, but in Daml changing a field archives the contract and creates a new one. So:

1. The contract carrying `ES_Announced` or `ES_Confirmed` is archived, and a new contract for the same `EventKey` is created carrying `ES_Corrected { supersededBy }`. Both remain in the update history; only the second is active.
2. The corrected event is created with its own `EventKey` and `supersedes` naming the original. Both records can be written in one transaction: a key is a value and can be named before the contract exists.
3. Affected entitlements are archived and re-created with `ETS_Superseded { supersededBy }`; the replacement entitlements carry `supersedes` naming the superseded key. Both records have the same observers, so a counterparty reading its own participant sees the archive, the create, and the link in both directions. No separate signal is needed.
4. **What has been paid carries over.** The replacement's `paidAmount` is the superseded record's `paidAmount`, and the superseded record keeps it too, as history. So the replacement's `outstanding = netAmount - paidAmount` is what is still owed under the corrected terms, and the compensating payment is an ordinary `Entitlement_Settle` on the replacement, with no special payment type and no extra `meta` keys.

The amounts, kept apart: the **original obligation** is the superseded record's `netAmount`; the **revised obligation** is the replacement's `netAmount`; the **amount already paid** is `paidAmount` on both; the **adjustment** is the difference of the two obligations; and the **outstanding** figure is the replacement's `netAmount - paidAmount`. An original obligation of 100, with 40 paid, corrected to 90, leaves 50 outstanding; it is not a recovery of 10. Repeated corrections chain the same way, each carrying `paidAmount` forward from the last.

**Overpayment is a claim, never a negative transfer.** Where `paidAmount` exceeds the revised `netAmount`, `outstanding` is negative: the registry has a claim on `payableTo` for the difference. The claim is represented by the replacement entitlement, whose figures state it, with the registry as creditor and the parties of `payableTo` as debtor. Nothing on the ledger enforces it, because the registry does not control the holdings it paid into. Recovery is the registry's own process. `Entitlement_Settle` rejects any non-positive amount, so no implementation can pay a negative figure by mistake. The suite tests that both directions are represented, not that recovery succeeds.

**A correction that changes `recordTime`, `kind` or `payInInstrument` is a cancellation, not a supersession.** Steps 3 and 4 work only when the superseded and new entitlements have the same observers and are denominated in the same instrument. A new record time entitles parties that were observers of nothing, and they would see a create carrying a `supersedes` key they cannot resolve. A new `kind` changes which redirects catch the event (§4.3), so the slices, their keys and their observers change. A new `payInInstrument` makes `paidAmount`, paid in one instrument, meaningless against a figure in another. So a registry correcting any of the three MUST cancel the event and its entitlements (`ES_Cancelled` / `ETS_Cancelled`) and announce a new event.

**When.** The registry cancels the event first, as soon as it decides on the correction; the event's `replacedBy` is set then, because the new event's key can be named before it exists. From that moment no entitlement of the event can be paid: `Entitlement_Settle` requires the event live (§4.2), and a cancelled event is not. The entitlements are then cancelled in batches sized to the synchronizer's request limits; an instrument with 50,000 holders cannot archive and re-create all its entitlements in one transaction. An entitlement's replacement cannot be named at cancellation, because which slices the new event has is known only after it derives. So the cancelled entitlements first carry `replacedBy = None`, and after the new derivation the registry re-creates each one that has a replacement, setting `replacedBy`, in the transaction that creates the replacement.

Consequences, because this case is not covered by the cancellation rule below:

- `ES_Cancelled` carries `replacedBy : Optional EventKey`, and `ETS_Cancelled` carries `replacedBy : Optional EntitlementKey`, so the link exists on-ledger in the forward direction for the parties who can see the cancelled record. The replacement event carries `supersedes = None`: it supersedes nothing, and reusing that field would give it two meanings.
- A cancelled entitlement names a replacement only where the new event has an entitlement for the same account and the same slice — the same `account` and `redirect` in the key. That replacement inherits the cancelled record's `paidAmount` where the two are paid in the same instrument, so money already moved counts toward the new figure. Where `payInInstrument` changed, it inherits nothing: the replacement starts unpaid, and the cancelled record's `paidAmount` is the registry's claim in the old instrument.
- A party entitled under the old record time and not under the new one sees its entitlement cancelled with no replacement. If it was already paid, the cancelled record's `paidAmount` is the registry's claim, by the overpayment rule above.
- A party entitled only under the new record time sees a first-time event and a first-time entitlement, with nothing on either marking it as a replacement; it could see neither cancelled record, so there is nothing on-ledger to link it to. The gap is intended: the context reaches it through the announcement endpoint (§8), which serves the cancelled event with its `replacedBy` alongside the new one.

Cancellation before payment: `ES_Cancelled`, entitlements `ETS_Cancelled`, nothing to compensate.

## 8. Off-ledger API

`servicing-v1.yaml`, at `/registry/servicing/v1/`. Shipped api-names drop the package's `token-` prefix, as `splice-api-token-transfer-instruction-v2` serves `/registry/transfer-instruction/v2/`. Reads are `GET` with pagination, as the metadata API's are; factory and choice-context endpoints are `POST` with a request body, as every shipped one is.

| Endpoint | Returns |
| --- | --- |
| `GET /registry/servicing/v1/instruments/{instrumentId}/events` | `ListEventsResponse`: `LifecycleEventView` values as plain data, filterable by status and time, with `pageSize` and a next-page token. Served as data: §4.1 keeps the event's observers narrow, so there is no contract to disclose. For display; the authoritative terms are on each entitlement |
| `POST /registry/servicing/v1/redirect-factory` | `RedirectFactoryWithChoiceContext`: the factory and its choice context in one response, as `POST …/transfer-factory` returns `TransferFactoryWithChoiceContext`. Builds the register first if it does not exist (§4.3) |
| `POST /registry/servicing/v1/redirect-instructions/{id}/choice-contexts/{approve\|reject\|withdraw}` | `ChoiceContext`. The approve context for the completing approval includes only the register as a disclosed contract: no holding reaches the beneficiary, and no cap is checked against the position on either side of the interface (§4.3) |
| `POST /registry/servicing/v1/redirects/{id}/choice-contexts/revoke` | `ChoiceContext`, carrying the register as a disclosed contract for a completing revocation. Needed because a revocation completes on a beneficiary-side call, and the beneficiary is an observer of the register only through disclosure. The shipped standard serves contexts for choices on a live object the same way: `/registry/allocations/v2/{allocationId}/choice-contexts/{withdraw\|cancel}`. `_Expire` needs none: the admin signs the register and reads it locally |
| `POST /registry/servicing/v1/entitlements/{ref}/choice-contexts/settle` | `ChoiceContext` with the servicing registry's own entries (the event contract for the liveness check and, for a redemption, the allocations), for a paying agent that is not the admin. The paying agent adds the cash registry's transfer-factory context to it (§6). Addressed by the opaque `ref`, so the path names no party |

Redirect instructions and live redirects sit under separate segments because both are addressed by id under one api-name, and an id alone would not say which of the two it names.

Source: `splice-api-token-metadata-v1/openapi/token-metadata-v1.yaml:13,31-63` (GET reads, pagination); `splice-api-token-transfer-instruction-v2/openapi/transfer-instruction-v2.yaml:13-120`; `splice-api-token-allocation-instruction-v2/openapi/allocation-instruction-v2.yaml:13-115`.

**No endpoint lists an account's entitlements.** The standard's off-ledger API serves factories and choice contexts; holdings are read from the Ledger API, and an entitlement whose observers are the account parties is readable the same way. A second source of truth for the same figures would recreate the reconciliation problem this proposal removes.

## 9. Privacy analysis

| Who | Learns | Already knew? |
| --- | --- | --- |
| Registry | holdings at record time, entitlements it derives, what it has paid | yes: it is the instrument admin and the payer |
| Registry, from a redirect | that a beneficiary exists, its identity, the scope and window | **new** |
| Holder's provider | its client's entitlements and receipts; every redirect on its client's position, with the beneficiary's identity and account | entitlements follow existing holdings visibility; beneficiary identity is **new** |
| Beneficiary | the entitlement payable to it: the pledged slice (`unitsAtRecord`), its gross, the deductions applied and their stated `source`, what has been paid, and `holderAccount`. Under an uncapped redirect the slice is the whole position, so the beneficiary learns the holder's position at every record time | **new**; the slice it already knows from the agreement, the deductions it did not |
| Beneficiary, on approving or revoking | the register's whole payload (`committedUnits`, `liveCount`, `uncappedLive`, `nextPriority`), because it is disclosed and a disclosed contract carries its full create argument. So: that other financing exists on the position, how much is committed ahead of it, and how many redirects have ever completed there. Never who and never on what terms. Not the position directly, but a beneficiary whose slice later falls short of its cap knows the position is what the redirects ahead of it took plus its own units, and from `committedUnits` it has a fair estimate of the first | **new**; aggregate, but enough to bound the position under a shortfall (§4.3, §5.1) |
| Beneficiary's provider | the holder's identity, account and pledged slice, as an account party of `payableTo`; and the register's payload, as the row above, whenever it is the beneficiary-side party that approves or revokes | **new** |
| Paying agent, where delegated | every entitlement it settles | **new**, and necessary to settle |
| Holder | its own entitlements and the redirects it authorized | — |
| Other holders of the instrument | nothing: the event's observers are narrow (§4.1), so the holder register is not disclosed | — |

**Five disclosures worth naming separately.**

1. **The deductions reach the beneficiary and its provider.** `Deduction` carries `rate`, `basis` and a free-text `source`, and they are observers of the entitlement, so a party with no relationship to the holder learns its withholding rate and the stated reason, which is a tax-residence and status signal. A registry that wants this narrowed can put the deduction detail only on the holder's own share and give the redirected slice the net figure; the cost is that the beneficiary can no longer check the deduction. That trade-off is part of the Milestone 1 gate.
2. **Both providers learn the other side.** The holder's custodian learns who financed its client; the beneficiary's custodian learns whose collateral it receives income from.
3. **`agreementRef` is a stable correlator.** It reveals nothing about the agreement's terms: the registry cannot fetch a contract in an application it has no visibility into. But the same `agreementRef` across several holders' redirects tells the registry those holders are financed under one facility, and the same id in any later disclosure links the two records. An application that wants the link broken should use a per-holder opaque reference and keep the mapping off-ledger.
4. **Timing.** Creation, `fromTime` / `toTime`, `priority` and revocation tell the registry when a financing opened and closed, per position, whether or not an entitlement is ever derived.
5. **The position, in two cases.** An uncapped redirect's beneficiary sees the holder's whole position at every record time, because its entitlement is the position. A capped redirect's beneficiary that receives less than its cap learns, with the register's `committedUnits`, roughly what the position was. A holder that wants its position kept from a financier uses a capped redirect set below the position.

**The Milestone 1 gate, with a pass/fail test.** The disclosures above are put in writing to at least one custodian and at least one buy-side holder, each asked whether it would permit a client position to be pledged under them, yes or no, with reasons. The answers are published whichever way they go. A "no" from either triggers the fallback.

**The fallback, and what it costs.** The beneficiary account is one owned by the holder's own provider: `owner = provider`, so `accountParties` reduces to `[provider]` and no financier is named on the ledger. An account merely held at that provider with `owner = beneficiary` would still name the beneficiary and would conceal nothing. Then: the registry learns that a provider-owned account received income, not who the financier is; the holder's provider learns everything, which it already did; there is no separate beneficiary's provider, because the provider is the payee. The provider onward-pays under its existing client agreement.

The financier then appears nowhere on the ledger, so the cross-provider case (the pledgee's income arriving at a second custodian, which reconciles its receipt to an entitlement it can read) is not demonstrated. The registry-against-provider comparison of §5.5 still is. The proposal's Funding section, under "Consequences of a miss", states the price.

## 10. The demonstration, precisely

Milestone 5's case, with every account and who sees what. Five parties, two of them providers:

```
                    Registry R  (instrument admin, paying agent)
                        │ signs holdings, events, entitlements, redirects, registers
        ┌───────────────┴──────────────────┐
  account A                          account B
  owner    = Holder H                owner    = Pledgee P
  provider = Custodian C1            provider = Custodian C2
        │                                  ▲
        └── redirect: coupons on up to N units of A ──┘
```

Five participants: R, H's wallet, C1, P and C2 each submit from their own, and each command acts as one party. The redirect is built in two to four commands, one party each: H proposes; C1 approves, if R's account policy asks for the provider; P approves; C2 approves if the policy asks, and whichever approval is last completes the redirect against A's register. A revocation is one command by P or C2; a request by H or C1 is recorded and waits for one of them.

| Party | Operates | Observes |
| --- | --- | --- |
| R | the registry application | everything it signs |
| C1 | a custody application on its own participant, which for the demonstration that evidences the standard is neither operated nor authored by PixelPlex | A's holdings, A's register, the redirect, both entitlements on A and the `paidAmount` on each as it is settled, and the holder's receipt into A |
| H | a wallet | the same as C1, as account party of A |
| P | a collateral application | the redirect, the redirected entitlement on A and its `paidAmount`, the receipt into B, the register by disclosure on approving or revoking |
| C2 | a custody application | the redirect and the redirected entitlement, as account party of B; the payment into B |

What is compared, and by whom: R against C1 on both entitlements on A (units, split, gross, deductions on a gross or remaining basis, net, payable account, and what was paid on H's own share; §5.5). These are the two independently operated applications the acceptance criterion names. P against R on what was paid on the redirected slice, against the receipt into B. P and C2 check only the redirected entitlement's arithmetic, because neither can see A's position.

The demonstration does not cover a multi-tier omnibus chain, where the registry sees only a custodian's aggregate account and the custodian allocates to its clients off-ledger. There the registry cannot derive a beneficial owner's entitlement, because it cannot see the beneficial owner; that sub-allocation is the custodian's own and out of scope. Here every account is at the registry, with its provider recorded on it, which is the custody relationship CIP-0112 makes explicit.

A redemption runs on the same accounts: the instrument frozen from `recordTime` minus the tolerance (§6), the pledge released and the units allocated from A to `cip-112/burn` by H or C1, and the cash to B where the redirect's `kinds` include `EK_Redemption` or are empty (§6).

## 11. Conformance suite

`splice-token-servicing-test`, in the form of the shipped `splice-token-standard-v1-test` / `-v2-test` packages, which a third-party registry copies and runs against its own templates.

`V2_VALIDATION.md` describes the initial validation of the V2 standard against the `token-standard-v2-daml-preview` branch and states that small refinements may still occur. It shows that this test-package form is what registries are asked to run.

Scenarios, each asserting field-by-field equality on the fields §5.5 marks derivable:

1. Coupon to a plain holder on a basic account (no provider).
2. Coupon to a pledged holding with a redirect, beneficiary at a different provider, the redirect built by one party per command (proposal, each approval and the completing approval each submitted as a single party, as from its own participant). A redirect approved by both providers but not by the holder account's principal does not complete, including where the beneficiary account is owned by the holder's provider.
3. Holder, account provider and beneficiary as three parties: the provider recomputes every field of every entitlement on the position, including the split.
4. Partial redirect via `unitsCap`: two entitlements for one account, `unitsAtRecord` summing to the position.
5. Two redirects from one account to the same beneficiary account: two entitlements with distinct keys, each naming its redirect.
6. A register exists for an account that has never had a redirect proposed, created by the registry's own command and not inside a transfer to the account. Two concurrent *completing approvals* of capped redirects on that position, including the first two on it: one commits, the other fails on an inactive contract and succeeds on retry; where one of the two is uncapped, the retry is rejected instead. Afterwards exactly one live register exists per position, with aggregates matching the §4.3 table.
7. A completion while an uncapped redirect is live, and an uncapped completion while any redirect is live: both rejected on-ledger, not retryable, including where the two redirects' kinds or windows do not overlap. An uncapped redirect proposed without `toTime`: rejected. Capped redirects whose caps together exceed the position: completed, because §4.3 does not check that, and then allocated by scenario 9.
8. A holder-side revocation call recorded as a request, the redirect still `RS_Live`; a beneficiary-side call completing it, asserting a successor carrying `RS_Revoked` and the register decremented; `_Expire` after `toTime`, asserting the register decremented and a successor carrying `RS_Expired`; an `_Expire` before `toTime`, and one on a redirect with `toTime = None`, both rejected.
9. A position that has fallen below the committed caps: the holder's share absorbs the shortfall first, the lowest-priority redirect second.
10. The snapshot: a redirect completed after `recordTime` with `fromTime` before it does not catch the event; a redirect revoked after `recordTime` does; holdings created and archived on each side of `recordTime` counted per §5.1.
11. Settlement: full payment; partial payment and then the rest; resubmitting a settlement that already committed fails and moves nothing; two concurrent settlements of one entitlement, one fails; a transfer that would complete as `Pending` aborts the settlement with nothing moved; an amount above `outstanding`, a non-positive amount, a partial settlement of a redemption, and a settlement by a party other than the paying agent, all rejected.
12. Redemption of a pledged holding: a transfer of the instrument after the record time rejected, and one whose ledger time falls inside the tolerance window before `recordTime` rejected too; the lock released and the holding allocated to `cip-112/burn` by one holder-side party; the cash to the pledgee under a redirect covering `EK_Redemption`, burned and paid in one transaction; and, separately, the cash paid before the allocation exists and the burn executed when it arrives.
13. Correction after partial payment: obligation 100, 40 paid, corrected to 90: the replacement carries `paidAmount` 40 and settles 50; the counterparty follows the supersession from its own participant.
14. Correction after full payment, overpayment direction: paid 100, corrected to 90: `outstanding` is −10, represented, and no transfer is made.
15. A settlement and a correction submitted concurrently on one entitlement: exactly one commits, and the outcome satisfies scenario 13 or 14.
16. Cancelled event before payment.
17. Corrections that change `recordTime`, `kind` or `payInInstrument`, after payment: each a cancellation with `replacedBy` set and a re-announcement; a replacement inheriting `paidAmount` where the payment instrument is unchanged and starting unpaid where it changed; and a cancelled entitlement with no replacement standing as a claim.
18. Holdings of the instrument found on a second synchronizer, or unassigned in flight, at the record offset: derivation refused.
19. Rounding: the §5.6 vectors, exactly; `decimals` outside 0..10 rejected; a product at 10^18 rejected.
20. Across parties: every entitlement of one event carries `terms` equal under the derived `Eq` (§4.0).

Pinned per Splice release in CI, the current one and the previous one.

## 12. Open questions for the Token Standards SIG

1. **Package namespace.** `splice-api-*` is the Splice/DA namespace; the name is the SIG's call. The draft uses `splice-api-token-servicing-v1` as a placeholder for a package of five interfaces.
2. **Does `recordTime` belong on the event, or does the standard want an explicit snapshot object?** We chose an instant plus the ACS-at-offset rule in §5.1. A snapshot contract would be auditable on-ledger but is O(holders) contracts per event, on top of §4.2's per-entitlement cost.
3. **Should entitlements be on-ledger contracts, or an off-ledger derivation with a signed digest on-ledger?** We chose on-ledger because the entitlement is what parties disagree about, a digest cannot be reconciled field by field, and a digest cannot be consumed atomically with a payment. §4.2's storage cost is the argument on the other side.
4. **Is a narrow-observer event plus denormalized `EventTerms` the right shape** (§4.1)? The choice is between a holder-register leak and duplicating terms on every entitlement, and duplication leaves one consequence we cannot remove from inside this package: no single party can check its copy against the source. A standard disclosure mechanism for instrument-level announcements would resolve it, and would be needed before an elections extension, where a holder acts on terms before any entitlement exists.
5. **Paying into an account that cannot receive directly.** `Entitlement_Settle` requires the movement to complete in its transaction (§6), so a `payableTo` whose payment instrument would answer `Pending` cannot be paid this way. Should the standard name a direct-receipt arrangement a beneficiary account must have, or define a settlement path through an allocation the beneficiary pre-authorizes?

## 13. Deliberate limits and extensions

**Extensions**, each a separate package on top of this one, reusing its keys, leaving its interfaces as they are, and carrying what it adds in interfaces of its own:

- **Holder elections**: cash or stock, participate or not. Needs typed option sets, quantity limits across elections, defaults, withdrawal and amendment, and what corrections do to a pending election; and the disclosure mechanism of §12.4, because a holder acts before any entitlement exists.
- **Unit changes**: splits and consolidations. An extension entitlement interface can state a unit result, but delivering it depends on whether the registry's holdings are admin-signed or jointly signed: in the standard's own reference token a holding is `signatory accountParties admin account, admin`, so the admin cannot re-issue unilaterally, and standardizing across both models needs a propose-accept path per holder.
  Source: `examples/splice-test-token-v2/daml/Splice/Testing/Tokens/TestTokenV2/Holding.daml:20-25`.
- **Partial redemption**, which needs a per-holding factor and a rule for which units are burned.
- **Residual allocation against a funded total**: a funded total on an extension event interface and a rule giving the rounding residual to holders. It needs a ranking every party can verify, which §5.4's per-entitlement locality cannot provide.
- **Securities-lending equivalent payments**: an obligation owed whatever the borrower's account holds (§4.3, "What a redirect is"), with its own source and debtor.
- **Mergers, spin-offs, rights issues**: events that create or exchange instruments.

**Limits of version 1:**

- **No recovery of overpayment**; the claim is represented. §7.
- **No tax computation.** §5.3.
- **No cross-synchronizer derivation**; the registry refuses rather than under-paying. §5.1.
- **No sub-allocation inside an omnibus account.** §10.
- **Redirect exclusivity is per position, not per scope.** §4.3.
- **Settlement only by a movement that completes in the transaction.** §6, §12.5.
- **Record date as a calendar convention is out of scope**; the ledger has instants. §4.1.

## 14. Risks

| Risk | Mitigation |
| --- | --- |
| The redirect disclosures are unacceptable to custodians or holders | Milestone 1 gate with a written yes/no test and a designed fallback, published either way, with a funding consequence stated in the proposal (§9) |
| Two implementations round differently and the standard forks | §5.2 pins the arithmetic to a single rounding, at the LF target the standard is built for; §5.4 keeps every figure local to its entitlement; vectors publish at Milestone 2 |
| A payment is made twice, or recorded without being made | settlement and its record are one transaction (§6); the suite's scenarios 11 and 15 exercise retry, concurrency and a racing correction |
| Per-entitlement storage cost deters adoption at scale | quantified in §4.2; the digest alternative is put to the SIG rather than assumed away |
| A beneficiary cannot receive directly, so its entitlement cannot be settled | stated as a limit and put to the SIG (§12.5); no unverifiable in-flight state is introduced to cover it |
| A flow needs parties hosted on different participants to act in one command | none does: every choice is callable by one party, and approvals accumulate on a pending instruction (§4.3); scenario 2 runs the redirect flow one party per command |
| The CIP-0112 Daml API is still in a validation phase | `V2_VALIDATION.md` states refinements may still occur; the package pins two Splice releases in CI |
| The SIG directs the design differently | milestones are adjusted with the Committee rather than delivered against a superseded draft |
| No independent adopter appears | a large share of funding is gated on Milestones 4–5, and the proposal states what is not paid if each is missed |
