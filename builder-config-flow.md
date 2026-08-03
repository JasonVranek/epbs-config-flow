# Per-key builder configuration: a walkthrough from the simplest config up

> **Status.** Reflects three open PRs at these commits: keymanager-APIs @`8da225a`,
> beacon-APIs @`97c1cd9` (#630), builder-specs @`36cefe4` (#165). These branches are still moving; when a
> spec changes, this doc is stale until updated. The specs are authoritative: where this doc and a spec
> disagree, the spec wins.

This is an implementer's guide to how a staker's builder preferences travel from a config file, through the
Keymanager API, into a validator client, out to a beacon node on block production, and finally to a builder.
The rules are spread across three specs, so this puts the semantics in one
place. It is an explainer, not a spec; where it and a spec disagree, the spec wins.

The three specs it traces:

- **Keymanager API**: how an operator programs a validator key's builder configuration.
- **Beacon API**: how the validator client hands that configuration to the beacon node when asking for a
  block (`produceBlockV4`), how it submits builder preferences for the beacon node to fan out
  (`submitBuilderPreferences`), and how it publishes the result (`publishBlockV2` for the signed block, plus
  the execution-payload-envelope endpoints).
- **Builder API** (`getExecutionPayloadBid`, `submitBuilderPreferences`, `submitSignedBeaconBlock`): how the
  beacon node talks to builders. Note `submitBuilderPreferences` exists on both APIs: the beacon-API call is
  the batched VC-to-BN hop, the builder-API call is the per-builder BN-to-builder hop it fans out into.

The doc is organized as a progression. It starts from the simplest possible configuration and layers complexity one example at a time. Each later example
explains only what is new relative to the one before it.

## Contents

- [The cast](#the-cast)
- [Example 1: local build only](#example-1-local-build-only)
- [Example 2: add p2p bids](#example-2-add-p2p-bids)
- [Example 3: add one builder-API connection](#example-3-add-one-builder-api-connection)
- [Example 4: multiple connections and fallbacks](#example-4-multiple-connections-and-fallbacks)
- [Example 5: edge-case configs](#example-5-edge-case-configs)
- [Open questions and out of scope](#open-questions-and-out-of-scope)
- [Appendix: more footgun rules](#appendix-more-footgun-rules)

## The cast

- **operator**: the human staker.
- **staking software**: whatever the operator runs to manage a node, for example a distributed-validator
  stack or a docker-compose setup. Not standardized; it owns a config file and speaks the Keymanager API.
- **validator client (VC)**: holds the keys, signs. Also called the **proposer** or **validator** in the
  context of proposing a block.
- **beacon node (BN)**: builds blocks, talks to builders, and selects the winning bid or its own build.
- **builder**: an external party that builds execution payloads and supplies them to the beacon node, either
  directly over a URL or gossiped over the p2p network. A URL usually fronts one builder, though it may front
  several behind an intermediary.

---

## Example 1: local build only

A key that only ever builds locally. To guarantee this regardless of how the validator client is globally
configured, set `enabled: false` at the keymanager; it overrides any global builder setup for this key:

```jsonc
// keymanager POST body
{ "enabled": false }
```

With builders disabled the validator client considers **no external bids at all**, neither over the builder
API nor over p2p, and the proposer builds its own block. None of the machinery in the later examples runs: no
bid requests, no selection among bids. At block time the VC realizes this by sending no `BuilderConfig` on
`produceBlockV4` (an omitted body).

---

## Example 2: add p2p bids

Now the proposer wants to consider bids seen over the p2p network, but still makes **no builder-API calls**.
Configure this at the keymanager with `enabled: true` and an empty `builders` list:

```jsonc
// keymanager POST body
{
  "enabled": true,
  "builders": [],                // no builder-API bids requested; p2p is the only source
  "min_bid": "10000000",         // floor for p2p bids (Gwei)
  "builder_boost_factor": "110"  // boost for p2p bids; 100 is 1.0x
}
```

An empty `builders` array requests no builder-API bids, so the only bids considered are those arriving over
p2p.

If you implemented the pre-Gloas `produceBlockV3`, note that `builder_boost_factor` was a query parameter
there; Gloas moves the bid-selection knobs into this `BuilderConfig` body.

### The floor and the boost

These two fields set how a bid's value is weighed against the local build:

- **The floor** (`min_bid`): the minimum **total** payment accepted, a bid's `value` plus its
  `execution_payment`. A bid below it is rejected. (A p2p bid's `execution_payment` is always `0`: consensus
  rejects a gossiped bid whose `execution_payment` is nonzero. So a p2p bid's total is just its `value`.)
- **The boost** (`builder_boost_factor`): a percentage multiplier on the surviving bid's total,
  `builder_boost_factor * (total // 100)`. `100` is the identity (1.0x); below 100 favors the local build,
  above 100 favors the bid. The highest boosted bid competes with the local build **in Gwei**, and the local
  build wins a tie.

---

## Example 3: add one builder-API connection

Now the proposer connects directly to one builder at its URL: the beacon node calls the builder at the
entry's `url` for a bid.

The keymanager body the staking software POSTs for one trustless direct connection:

```jsonc
// keymanager POST body
{
  "enabled": true,   // false = the VC uses no external bids for this key
  "builders": [
    { "url": "https://builder.example.com",
      "max_execution_payment": "0" }   // 0 = accept no trusted payment; whole payment must be trustless
  ]
}
```

The staking software `POST`s this `BuilderConfig` to the VC at `/eth/v1/validator/{pubkey}/builders` (`202`). The submission **replaces** the key's
stored config in full; the server does not merge with what was there before. Note what is absent from the
entry: no `auth_data`, no `builder_pubkey`, no per-entry `min_bid` or `builder_boost_factor`. These are
optional in the keymanager body, and the VC resolves all but `builder_pubkey` before anything reaches the
beacon node.

### Tracing the three interfaces

1. **Keymanager POST** (above): the operator programs the key; the VC stores the config.
2. **The VC resolves every entry field.** Beacon-side entries are fully resolved: on `produceBlockV4` every
   field is present. The VC fills in `min_bid`, `builder_boost_factor`, `max_execution_payment`, and
   `auth_data` from its own configuration where the entry omitted them.
3. **Beacon `produceBlockV4`.** The VC calls `produceBlockV4` with the resolved `BuilderConfig` in the body.
   For each entry the BN calls `getExecutionPayloadBid` at that entry's `url`, forwarding the entry's
   `auth`.

### Request authentication

A request auth lets a builder confirm it is talking to the actual proposer over a direct builder-API
connection.

Tracing the field down the stack: the keymanager body may leave `auth_data` unset (the example above does),
but the builder-API requires the bid request to be authenticated, and a beacon-side entry is fully resolved.
So by the time an entry reaches `produceBlockV4` it carries its `auth`. The VC bridges the two layers: it
resolves `auth_data` and constructs the signed `auth` before the beacon call.

The VC builds and signs the `auth` (a `SignedRequestAuth`, a `message` of `{data, slot}` plus a
`signature`) per entry, per slot; the BN forwards it byte-for-byte and does not sign. Two senses of
"fork-versioned" pull apart here and both are worth stating up front: the **signing domain** is not
fork-versioned (it uses genesis defaults, below), but the **wire type** is, so any request that carries the
`auth` also carries an `Eth-Consensus-Version` header naming the fork whose `RequestAuth` schema it encodes:

```python
def sign_request_auth(auth_data, slot, validator_privkey):
    # signing domain: genesis fork version + zero root (NOT fork-versioned); never reuse DOMAIN_BEACON_BUILDER
    domain = compute_domain(DOMAIN_REQUEST_AUTH)
    message = RequestAuth(data=auth_data, slot=slot)
    signing_root = compute_signing_root(message, domain)
    return bls.Sign(validator_privkey, signing_root)
```

`auth_data` is opaque, agreed with the builder out of band. It should be unique to the builder; it can be
anything agreed out of band, and when nothing is agreed the sensible default is the builder's exact URL,
because the URL uniquely identifies the entity you are sending the builder-API call to. So when an entry omits
it, the VC derives it from the builder's URL by the SHOULD convention: the UTF-8 bytes of the URL **exactly as
advertised**, hex-encoded. The VC signs that derived value and the builder byte-matches it, so the two must
produce the byte-identical string.

### Communicating preferences

The first place the auth is used is `submitBuilderPreferences`, sent ahead of the proposal slot so the
builder has time to apply the preferences to its bid (for example, an epoch before). It travels in two hops,
and the same name is used for both:

- **VC to BN, batched (beacon API).** The VC POSTs a flat array of `BuilderPreferencesEntry` to
  `/eth/v1/validator/builder_preferences`, one entry per builder per proposer, each naming the proposer it
  applies to in its `proposer_pubkey` field. `BuilderPreferencesEntry` is a dedicated type —
  `{proposer_pubkey, url, max_execution_payment, auth}` — distinct from the block-production `BuilderEntry`.
  Every entry carries a `url` (the builder it targets), and the `Eth-Consensus-Version` header is required. The BN responds `200` when every entry was submitted and its
  builder accepted it, or `400` with an `IndexedErrorMessage` that reports each failed entry by its index,
  proxying the builder's error for that entry, while still submitting the others.
- **BN to builder, per-builder (builder API).** For each entry the BN makes one builder-API
  `submitBuilderPreferences` call to `.../builder_preferences/{proposer_pubkey}` — the proposer pubkey rides
  in the **path** (symmetric with `getExecutionPayloadBid`), not the body. The body is a
  `BuilderPreferencesRequest` of just `{auth, preferences}`, where `preferences` carries the
  `max_execution_payment`. Each addressed builder responds `202`, `400`, or `401` individually and
  best-effort, and the BN maps those results back onto the batched response above.

The auth binds each preference request to the real proposer, not an imposter. Its `auth.message.slot` is the
proposal slot the preferences apply to, and the builder rejects a submission whose slot has **already
passed** (`400`), which stops a replay from rolling preferences back. Signature failure is `401`, a `data`
mismatch is `400`.

For v1, the only preference it carries is `max_execution_payment`, the cap on the **trusted** portion of a bid
(`execution_payment`). `"0"` accepts no trusted payment, requiring the whole payment to be trustless;
`MAX_EXECUTION_PAYMENT` (`2**64 - 1`) accepts any amount. The cap has two channels:

- **Communicated to the builder**, which, once it has stored preferences, MUST NOT return a bid whose
  `execution_payment` exceeds the stored `max_execution_payment`. Without stored preferences it MAY bid any
  amount.
- **Backstopped at the BN**, which discards any returned bid over the configured cap regardless. So a builder
  that never received preferences still cannot get an over-cap bid accepted.

### Requesting a bid

With preferences in place, the BN requests a bid on the proposer's behalf. `getExecutionPayloadBid` is a
`POST` to `.../execution_payload_bid/{slot}/{parent_hash}/{parent_root}/{proposer_pubkey}` with the
`SignedRequestAuth` as the body and three **required** headers: `Eth-Consensus-Version` (the fork whose
`RequestAuth` schema the body uses, since the type is fork-versioned), `Date-Milliseconds` (the Unix ms the
request was sent), and `X-Timeout-Ms` (the proposer's timeout, measured from `Date-Milliseconds`). The builder is
expected to honor its stored preferences, and MUST respond by `Date-Milliseconds + X-Timeout-Ms`; a later
response is discarded, so a slow bid is a lost bid even at `200`. The builder:

- returns a bid (`200`) or, if it has none, `204`;
- returns `401` if the auth **signature** does not verify against `proposer_pubkey`;
- returns `400` when `auth.message.data` does not match the agreed value, when `auth.message.slot` does not
  equal the path `slot`, when a required timing header is missing, when the parent hash is unknown, or when
  the body is missing or malformed.

### The bid math

For each candidate bid the BN applies, in order: the `min_bid` floor (on the total), the
`max_execution_payment` cap (on the trusted portion), then the `builder_boost_factor` multiplier (on the
total). Worked in Gwei for a builder configured with `min_bid = 10000000`, `max_execution_payment =
200000000`, `builder_boost_factor = 110`:

- **Bid X**: `value = 5000000`, `execution_payment = 3000000`. Total = 8000000, below the floor of 10000000,
  so **rejected** before boosting.
- **Bid Y**: `value = 100000000`, `execution_payment = 250000000`. `execution_payment` 250000000 exceeds the
  cap of 200000000, so **rejected**.
- **Bid Z**: `value = 100000000`, `execution_payment = 50000000`. Total = 150000000 is at or above the floor,
  and 50000000 is at or below the cap, so it passes both gates. Boosted: `110 * (150000000 // 100) =
  165000000`. That is what competes with the local build's value.

```python
def evaluate(bid, min_bid, max_exec_payment, boost):
    total = bid.value + bid.execution_payment
    if total < min_bid: return None                           # floor, on total
    if bid.execution_payment > max_exec_payment: return None  # cap, on trusted portion
    return boost * (total // 100)                             # boost, on total (divide first)
```

### `builder_pubkey` filters the response

The `builder_pubkey` filter is optional. If an entry carries one, it **filters the response**: a bid that
comes back not signed by the expected builder MUST NOT be accepted. A correctly-signed bid from an unexpected
builder is dropped, not an error. This matters because you are allowing a trusted payment: if you want to bound
that trust to a specific entity, it may extend only to a specific builder and not to the URL, since one URL can
front several builders and you may trust only some of them.

Three builder-side MUSTs on the bid relate to the config values: `fee_recipient` MUST equal the fee recipient
from the proposer's `ProposerPreferences` (learned from the `proposer_preferences` gossip topic, which the
builder SHOULD subscribe to) whether the builder pays via `value` or `execution_payment`; and
`execution_payment` MUST NOT exceed the stored `max_execution_payment`. The two payment channels differ in
what they guarantee: `value` is committed, drawn from the builder's staked collateral; `execution_payment` is
only a promise the proposer trusts the builder pays them as part of the block.

### The bid-win path

If this builder's bid wins the auction, the block reaches the builder in a linear sequence:

1. `produceBlockV4` returns the winning bid to the VC as its output (only the `BeaconBlock`, since a bid won),
   along with an `Eth-Builder-Url` naming the winning builder.
2. The VC signs the block and calls `publishBlockV2` (`POST /eth/v2/beacon/blocks`), echoing `Eth-Builder-Url`
   in the request.
3. The beacon node both gossips the block to the network AND calls the winning builder's
   `submitSignedBeaconBlock`, routed by that echoed `Eth-Builder-Url`. The builder validates the
   `SignedBeaconBlock` (`400` if invalid) and, on `202`, releases the execution payload by constructing and
   gossiping the `SignedExecutionPayloadEnvelope`, assisting dissemination.

`Eth-Builder-Url` is how the beacon node knows which builder to forward to: the `produceBlockV4` response names
the winning builder's URL, the VC echoes it on publish, and the beacon node forwards there. It is absent when
the block was self-built or a p2p bid won. Echoing the URL means the VC can publish through any beacon node,
not only the one that ran the auction, and that node can still forward to the winning builder, which keeps the
bid-win path stateless for multi-BN and failover setups.

---

## Example 4: multiple connections and fallbacks

Now several entries at once, with per-entry overrides. Only the diffs from Example 3 follow.

### Uniqueness rules

No two entries may share both the same `url` **and** the same `auth_data` (compared as decoded bytes, so
hex case does not distinguish, and an omitted `auth_data` is compared as the value the VC would derive).

A `url` can be an intermediary that fronts several builders rather than a builder's own address, with the
`auth_data` carrying the routing that selects which builder behind it a request is for. Reaching them all
through the one `url` means several entries have to share it, so an entry is keyed on `(url, auth_data)`, not
on `url` alone. Two entries **MAY** therefore share a `url` with different `auth_data`; the uniqueness rule
only forbids sharing both, which would just send the identical request twice.

```jsonc
// keymanager builders
"builders": [
  { "url": "https://a.example.com", "auth_data": "0x...a", "builder_pubkey": "0xB..." },
  { "url": "https://a.example.com", "auth_data": "0x...b", "builder_pubkey": "0xC..." },  // ok: same url, different auth_data
  { "url": "https://a.example.com", "auth_data": "0x...a", "builder_pubkey": "0xB..." }   // 400: duplicate (url, auth_data) of the first
]
```

### Per-entry vs per-config `min_bid` and `builder_boost_factor`

`min_bid` and `builder_boost_factor` keep the same two names at two structural scopes; the scope tells you
which value you are looking at. Shown inline:

```jsonc
// keymanager POST body
{
  "enabled": true,
  "builders": [
    { "url": "https://builder-b.example.com",
      "builder_pubkey": "0xB...",
      "min_bid": "20000000",            // per-entry: applies to B's builder-API bid
      "builder_boost_factor": "120" },  // per-entry: applies to B's builder-API bid
    { "url": "https://builder-d.example.com" }  // no per-entry knobs; inherits the per-config defaults below
  ],
  "min_bid": "10000000",                // per-config default (see Inheritance and defaults)
  "builder_boost_factor": "100"         // per-config default (see Inheritance and defaults)
}
```

A per-entry value applies to that entry's bid. The top-level per-config value is the key's default for that
field, used two ways: an entry that omits its own value inherits it, and p2p bids take it.

### Inheritance and defaults

The VC resolves every omitted field before the body reaches the beacon node. An omitted field inherits a
default, and what it inherits from differs by field, because `BuilderConfig` carries a key-level default for
only two fields:

- `min_bid`, `builder_boost_factor`: an omitted entry value inherits this key's `BuilderConfig` default, and
  if that is unset too, the VC's own configuration.
- `max_execution_payment`, `auth_data`: an omitted entry value inherits the VC's own configuration directly;
  there is no key-level default for these.

The key-level default is the footgun: if an operator sets a key-level `builder_boost_factor` but omits it on
an entry, the entry inherits the **key default**, not the VC's global. "I left it blank" does not mean "use
the client default."

```jsonc
// keymanager POST body
{
  "enabled": true,
  "builders": [
    { "url": "https://builder-b.example.com",
      "builder_pubkey": "0xB..." }   // omits builder_boost_factor -> inherits 120 (the key default), not the VC global
  ],
  "builder_boost_factor": "120"      // key-level default
}
```

### A bad body fails, a bad entry does not

A **body** that cannot be decoded (malformed JSON, or SSZ that fails deserialization) is a `400`; in SSZ the
whole `List[BuilderEntry]` either decodes or it does not, with no partial recovery. A single **entry** the BN
cannot use yields no bid but MUST NOT fail the request, so one bad entry never costs the proposer its slot.

---

## Example 5: edge-case configs

### Omitted and explicit-empty `auth_data`

When omitted, `auth_data` is VC-derived from the URL (the SHOULD convention from Example 3: UTF-8 bytes of the
URL exactly as advertised, hex-encoded). "Exactly as advertised" is the canonicalization rule (no
normalization); any divergence between what the VC signs and what the builder expects is a `400` at the
builder.

Two things the spec does not fully pin down, so do not over-rely on them. An **explicit** empty `auth_data`
(`"0x"`, which the pattern permits) is a distinct value in the equality comparison, but the builder requires
authentication on every request, so an empty value is likely rejected downstream. Treat "omitted" (VC derives)
and "explicit empty" as different, and prefer omitting.

The URL-derived default is the **same** for every builder behind a shared URL, so it cannot tell them apart. To handle that case the operator MUST set an explicit, distinct `auth_data`
per entry, agreed out of band. The URL-derived default is enough only when the URL fronts a single builder.

### SSZ absence sentinels

In JSON an optional field is simply absent. SSZ has no absence: every field in the container is always
present, so unset must be a sentinel value: `builder_pubkey` is all zero (not a valid BLS key).

### `enabled: false`, `DELETE`, and the four ways to say "fewer builders"

Four distinct states, not synonyms:

```jsonc
// four keymanager requests, one per state
{ "enabled": true }                   // omit builders: follow the VC's global config
{ "enabled": true, "builders": [] }   // builders: []:   no builder-API bids, p2p only
{ "enabled": false }                  // enabled: false: no builder bids at all
// DELETE /eth/v1/validator/{pubkey}/builders  (no body): remove the config
```

- **omit `builders`**: this key follows whatever builders the VC is globally configured with.
- **`builders: []`**: this key uses no builder-API builders; p2p bids remain its only source (Example 2).
- **`enabled: false`**: this key sources **no** builder bids at all, including from any builders the VC is
  globally configured with. This is the guaranteed local-build config from Example 1.
- **`DELETE`** the config: remove it entirely; the key reverts to the VC's own configuration, exactly as if it
  had never been configured. A `DELETE` is the absence of a stored config, where `enabled: false` is a stored
  one.

### Reading the config back

`GET /eth/v1/validator/{pubkey}/builders` returns the configuration **in effect** (`200`), with omitted values **resolved**: an entry that omitted a field
comes back with the value that will actually be used. This matters for read-modify-write. If you `GET`, tweak
one field, and `POST` the result back, every field the VC resolves is now an **explicit** value, and those
entries no longer track the defaults. To keep an entry tracking the key default, omit the field on the way
back in; do not echo the resolved value.

---

## Open questions and out of scope

Not everything the flow touches is settled or covered here. Genuinely open, still being worked in the specs:

- **Cap granularity.** `max_execution_payment` is per-entry (so per-URL), but `submitBuilderPreferences` is
  per-proposer-key. Which cap is communicated to a builder reachable at two URLs with two different caps is not
  pinned; the per-entry cap is the authoritative BN backstop regardless.
- **The local build's value.** Selection has a bid "compete with the local build" and the local build win a
  tie, but how the BN derives the local build's value (and in what units it compares) is a BN-internal the
  beacon spec does not pin.
- **Boost overflow and builder-vs-builder ties.** The boost's overflow bound (saturate to what type?) and the
  tie-break between two equal top *builder* bids are not pinned; only the local-vs-builder tie is.

Deliberately out of scope: consensus-spec bid construction and validity (`is_eligible_for_bid`, `gas_limit`,
collateral coverage), the PTC and gossip mechanics, blobs and KZG, the full contents of `ProposerPreferences`,
and the security/replay properties of the request auth and preferences messages (raised separately with the
spec authors).

## Appendix: more footgun rules

The rules most easily gotten wrong:

| # | Rule | Why it matters | Where |
| --- | --- | --- | --- |
| 1 | Resolution is 3-tier for `min_bid`/`builder_boost_factor` (entry, key default, VC config), 2-tier for `max_execution_payment`/`auth_data` | An entry that omits a field inherits the **key** default, not the VC global; "I left it blank" does not mean "use the client default" | keymanager |
| 2 | Omitted `auth_data` is VC-derived from the URL and byte-matched at the builder | The URL-derived value is identical for every builder behind a shared URL, so it cannot tell them apart; set an explicit, distinct `auth_data` per builder | keymanager, builder |
| 3 | SSZ absence sentinel: all-zero `builder_pubkey` | SSZ has no absence, so unset must be a sentinel value, and the JSON and SSZ forms must agree on what an entry means | beacon |
| 4 | Top-level `min_bid`/`builder_boost_factor` apply to p2p bids | There is no top-level `max_execution_payment` because a p2p bid carries no trusted `execution_payment` (consensus forces it to `0`) | beacon |
| 5 | Request auth: genesis **signing domain**, fork-versioned **wire type** | Sign under `compute_domain(DOMAIN_REQUEST_AUTH)` with genesis defaults (never the active fork version, never `DOMAIN_BEACON_BUILDER`), yet the SSZ type is fork-versioned, so `getExecutionPayloadBid` and `submitBuilderPreferences` require the `Eth-Consensus-Version` header | builder |
