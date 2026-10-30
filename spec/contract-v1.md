# Manometer proposal contract v1

The same document as https://manometer.bet/developers.html.

## Overview

Everything you submit is a **proposal**, never an order. The operator decides, per source, whether, where and with what stake a proposal is executed. Your stake hint and your claimed expected value are advisory and never binding.

- **Expected value is recomputed** on our side from your fair price against the real executable bookmaker price. Systematically overselling your edge is measured as *EV honesty* and costs rating.
- **Every proposal gets a receipt**: accepted, or rejected with a reason, and later scored with closing-line value and paper profit. That feedback loop is how you qualify.
- **New sources start in shadow**, paper only. Promotion to real money is earned with closing-line value, not with short-term profit.

The referee address for the public network is `https://app.manometer.bet`. Scorecards live at `https://manometer.bet/s/<sourceId>`.

## Quickstart

1. **Get a source id and an API key.** Apply as a signal source; you start in shadow.
2. **Send one proposal in clear** to see the round trip. The answer is your signed receipt.
3. **Seal the next ones.** Encrypt to the drand round of kick-off and send only the ciphertext.
4. **Keep your receipts.** Any receipt can be fetched again with its anchor path from `/netz/quittung/<rid>`.
5. **Preview your scorecard** and publish it when you are ready, or never.

```
# clear: scored at receipt, readable at once
curl -X POST https://app.manometer.bet/netz/submit \
  -H "Content-Type: application/json" \
  -d '{
  "sourceId": "src_you",
  "apiKey": "YOUR_API_KEY",
  "proposalId": "you-2026-11-02-0001",
  "proposal": {
    "proposalId": "you-2026-11-02-0001",
    "sourceId": "src_you",
    "sport": "fb",
    "mkt": "1x2",
    "sel": "2",
    "eventIdentity": {
      "home": "Brann",
      "away": "Molde",
      "league": "Eliteserien",
      "startsTs": 1793642400000
    },
    "fairOdds": 2.6,
    "devigMethod": "shin",
    "minOdds": 2.7,
    "tsCreated": 1793266867318,
    "ttlMs": 120000,
    "stakeHint": 10,
    "ev": 6.5
  }
}'
```

The answer is `200 {"ok": true, "receipt": {…}}`. Sending the same proposal id again returns the same receipt with `"duplicate": true`: exactly one receipt per id, ever.

## Authentication

Each source has a `sourceId` and an `apiKey`. Both go in the JSON body of every request, **never in the URL**. The key is never echoed back, never written into any file the source can read, and can be rotated in the console. Repeated failures from one address are throttled.

## Endpoints

| Method | Path | Access | What it does |
| --- | --- | --- | --- |
| `POST` | `/netz/submit` | source | Submit a proposal, clear or sealed. Answers with the signed receipt. |
| `GET` | `/netz/pubkey` | public | The referee's Ed25519 key, its key id, and the drand chain parameters. |
| `GET` | `/netz/quittung/<rid>` | public | One receipt and, once anchored, its Merkle path to the hourly anchor. |
| `GET` | `/s/<sourceId>` | public | The scorecard page. Data only when the source has published. |
| `GET` | `/s/<sourceId>/daten.json` | public, if published | Scorecard data. |
| `GET` | `/s/<sourceId>/bundle.json` | public, if published | The full bundle the verifier checks. |
| `POST` | `/netz/vorschau` | source | Preview your scorecard while it is private. |
| `POST` | `/netz/veroeffentlichen` | source | Publish or withdraw your scorecard. |
| `POST` | `/netz/bundle` | source | Download your full bundle while private. |

Route names are part of contract v1 and will not change within it.

## Proposal contract

Required fields. The validation source of truth is the platform's own validator; the machine-readable schema is in the console.

| Field | Type | Meaning |
| --- | --- | --- |
| `proposalId` | string | Your unique idempotency key. Re-submits with the same id are deduplicated. |
| `sourceId` | string | Your assigned source id. Must equal the envelope's. |
| `sport` | string | `fb` · `basket` · `tennis` · `cs2` |
| `eventIdentity` | object | `{ home, away, league, startsTs }`. Team names as plain strings. `startsTs` is kick-off in ms UTC and required. Add `pinId`, the Pinnacle matchup id, if you have it: it makes the match exact. |
| `mkt` · `scope` · `line` · `sel` | — | Market, see the taxonomy. `scope` defaults to `match`. |
| `fairOdds` | number > 1 | Your no-vig fair price for the selection. |
| `devigMethod` | string | How the fair price was derived, e.g. `shin`, `proportional`, `model`. Required: expected values are not comparable without it. |
| `minOdds` | number > 1 | Your slippage floor. Below it, the proposal is rejected with *line moved*. |
| `tsCreated` · `ttlMs` | number | Creation time in ms UTC, and time to live. Value decays; expired proposals are rejected. |

```
{
  "proposalId": "you-2026-11-02-0001",
  "sourceId": "src_you",
  "sport": "fb",
  "mkt": "1x2",
  "sel": "2",
  "eventIdentity": {
    "home": "Brann",
    "away": "Molde",
    "league": "Eliteserien",
    "startsTs": 1793642400000
  },
  "fairOdds": 2.6,
  "devigMethod": "shin",
  "minOdds": 2.7,
  "tsCreated": 1793266867318,
  "ttlMs": 120000,
  "stakeHint": 10,
  "ev": 6.5
}
```

### Optional fields

| Field | Type | Meaning |
| --- | --- | --- |
| `stakeHint` | number | Recommended stake in EUR. Advisory. |
| `ev` | number | Your claimed expected value in %. Logged and compared with ours. |
| `confidence` | 0–1 | Your own confidence. |
| `rationale` | string | Free text or tags: "steam move", "model edge". |
| `clvRef` | string | Closing-line reference to be graded against. Default: the sharp close. |
| `originOdds` · `originBook` | number · string | Where you saw the price, for a plausibility check. |
| `bookQuotes` | array | Odds for the same selection at other bookmakers, e.g. `[{"book":"stoiximan","odds":2.15}]`. We recompute expected value per bookmaker and route to the best one we can place on. |
| `live` · `endsTs` | boolean · number | In-play proposal, see below. |
| `bestOf` | number | Series length for CS2 and tennis: 1, 3 or 5. |
| `otIncluded` · `settleRule` | — | Basketball: overtime included? Tennis: retirement rule. |

### Market taxonomy

Handicap lines are from the home side's perspective; away is the mirror.

| Sport | mkt | scope | line unit | sel |
| --- | --- | --- | --- | --- |
| **fb** | `1x2` | match | — | 1 · X · 2 |
| `ah` | match | goals, quarter lines allowed | home · away |
| `total` | match | goals | over · under |
| **basket** | `ml` | match | — | home · away |
| `spread` | match | points | home · away |
| `total` | match | points | over · under |
| **tennis** | `ml` | match | — | home · away |
| `setHc` | match | sets | home · away |
| `gamesHc` | match | games | home · away |
| `gamesTotal` | match | games | over · under |
| **cs2** | `ml` | match | — | home · away |
| `mapHc` | match | maps | home · away |
| `mapsTotal` | match | maps | over · under |
| `mapWinner` | map1 · map2 · map3 | — | home · away |
| `roundsAh` | map1 · map2 · map3 | rounds | home · away |
| `roundsTotal` | map1 · map2 · map3 | rounds | over · under |

CS2 money lines are two-way; there is no draw selection. The referee scores the markets for which the board has a Pinnacle fair price; see scoring rules.

### In-play proposals

A pre-match proposal at or after kick-off is rejected. If your signal is in-play, set `"live": true`: then `startsTs` is still the kick-off, but now has to be in the past. You supply your own live fair price; a stale pre-match fair is never substituted. Live fair prices decay in seconds, so the maximum age is 15 seconds instead of 120. Add `endsTs` so that a decided segment cannot be bet. Live is enabled per source by the operator; until then, live proposals are rejected with *live signals not enabled for this source*.

## Sealing

While you qualify, you do not have to show us your picks. Encrypt the proposal JSON with **tlock** to the **drand quicknet** round of kick-off and send only the ciphertext. Nobody can open it before that round, the operator included. At the round, the public drand beacon opens it and the referee scores it as of the second it was received, against the market snapshot it took then.

- A seal that opens more than 60 seconds before kick-off is scored but **not executable**. One that is already open when it arrives is refused with 400.
- **All or nothing:** every seal appears on your scorecard. A seal that does not open to a valid JSON proposal counts as an invalid seal.
- Kick-off is taken from the matched board row (Pinnacle), otherwise from your `eventIdentity.startsTs`.
- The proposal id and source id inside the seal must equal the envelope's; a sealed proposal whose inside differs is not scored.

```
// sealed: Node 18+, npm install tlock-js@0.9.0. Encryption needs no network.
const { timelockEncrypt, Buffer } = require("tlock-js");
const QUICKNET = { hash: "52db9ba70e0cc0f6eaf7803dd07447a1f5477735fd3f661792ba94600c84e971",
  public_key: "83cf0f2896adee7eb8b5f01fcad3912212c437e0073e911fb90022d3e760183c8c4b450b6a0a6c3ac6a5776a2d1064510d1fec758c921cc22b0e17e63aaf4bcb5ed66304de9cf809bd274ca73bab4af5a6e9c76a4bc09e76eae8991ef5ece45a",
  period: 3, genesis_time: 1692803367, schemeID: "bls-unchained-g1-rfc9380", metadata: { beaconID: "quicknet" } };
const chain = { chain: () => ({ info: async () => QUICKNET }) };
const roundAt = (ms) => Math.ceil((ms / 1000 - QUICKNET.genesis_time) / QUICKNET.period) + 1;

const proposal = {"proposalId":"you-2026-11-02-0001","sourceId":"src_you","sport":"fb","mkt":"1x2","sel":"2","eventIdentity":{"home":"Brann","away":"Molde","league":"Eliteserien","startsTs":1793642400000},"fairOdds":2.6,"devigMethod":"shin","minOdds":2.7,"tsCreated":1793266867318,"ttlMs":120000,"stakeHint":10,"ev":6.5};
const round = roundAt(proposal.eventIdentity.startsTs);       // 33,613,012: the first round at or after kick-off
const ciphertext = await timelockEncrypt(round, Buffer.from(JSON.stringify(proposal)), chain);

const res = await fetch("https://app.manometer.bet/netz/submit", {
  method: "POST", headers: { "Content-Type": "application/json" },
  body: JSON.stringify({ sourceId: "src_you", apiKey: "YOUR_API_KEY", proposalId: proposal.proposalId, sealed: { ciphertext } }),
});
const { receipt } = await res.json();                        // keep it: it is your proof
```

## Receipts

Every accepted proposal gets a receipt signed with the referee's Ed25519 key (`GET /netz/pubkey`).

```
{
  "v": 1,
  "typ": "manometer.receipt",
  "sourceId": "nordic-models",
  "proposalId": "nm-2026-10-29-0412",
  "receivedTs": 1793266867318,
  "sealed": true,
  "payloadSha256": "56b41a5562e12c10f0a530a294f12f68b0f3245a40de94abc67059253dfbd04d",
  "drand": {
    "chain": "52db9ba70e0cc0f6eaf7803dd07447a1f5477735fd3f661792ba94600c84e971",
    "round": 33613012,
    "opensTs": 1793642400000
  },
  "snapshot": {
    "id": "snap_20261029T094105Z",
    "root": "6b53b9a0155cd54029ae5bc4e62f0ea070faa060e9cc56a5dddb924961438c16",
    "ts": 1793266865178,
    "n": 1458
  },
  "referee": "manometer-referee/1",
  "rid": "r_201f6283f23fdd7e55df6902",
  "keyId": "02d02ca9b91687c8",
  "sig": "ebtcqCM5gyuLQ7xAlABvggeDUy/O9HEHDLQYT7LaIwSYfLkpwXoN8MexF1JLZ4UAedjLd1KJlCemZ91LKudaDg=="
}
```

- **Canonical JSON**: object keys sorted, no whitespace, undefined dropped.
- `rid` = `r_` + the first 24 hex characters of SHA-256 over the canonical receipt *without* `rid`, `keyId` and `sig`.
- `keyId` = the first 16 hex characters of SHA-256 over the raw 32-byte public key.
- `sig` = Base64 Ed25519 signature over the canonical receipt without `sig`.
- `payloadSha256` = SHA-256 of the ciphertext (sealed) or of the canonical proposal (clear).
- **Market snapshot.** Because a sealed proposal cannot be read on arrival, the referee freezes the whole pre-match board of that second: Pinnacle no-vig fair and every bookmaker's prices for 1X2, money lines, and the handicap and total lines that have a fair. Its Merkle root (RFC 6962) is in the receipt; after opening, your bundle carries the one relevant row with its path.
- **External time anchor.** Every hour the referee builds one Merkle root over all new receipts, closing lines and results and submits only that 32-byte hash to the public OpenTimestamps calendars, which commit it into Bitcoin. The `.ots` proof is in the bundle and checks with the official `ots verify` as well.

`GET /netz/quittung/<rid>` returns `{ ok, receipt, anchor }`, where `anchor` is `{ anker, index, n, wurzel, pfad }`: the leaf is SHA-256(0x00 ‖ canonical signed receipt), the path runs from the bottom up, and it is `null` until the next hourly anchor. You can check any receipt in your browser.

## Lifecycle

```
submitted → valid → enriched → proposed → accepted | rejected | expired
accepted → placed → settled → graded
```

Every terminal state produces a receipt, and a rejection always carries its reason:

| Reason | Meaning |
| --- | --- |
| **line moved below originator minOdds** | The executable price fell under your floor. |
| **stale signal** | Older than the per-source maximum age, 120 seconds by default. |
| **expired** | The time to live ran out before processing. |
| **event already started** | A pre-match proposal at or after kick-off. For in-play, set `live: true`. |
| **live signals not enabled for this source** | The operator has not enabled in-play for your source yet. |
| **live, but not started yet or already finished** | In-play before kick-off, or after `endsTs`. |
| **real EV below policy minimum** | The recomputed expected value, not your claim, is under the source's threshold. |
| **odds outside policy band or caps** | Platform-side risk limits: odds band, open bets, daily exposure. |
| **no executable book odds or no match** | The event or market cannot be resolved at an execution bookmaker. |

The graded receipt adds the real placed odds and stake, the closing no-vig price, closing-line value, the result and the profit. That is the data your rating is built from.

## Scoring rules

- **Executable** = valid at receipt (not expired, before kick-off), not older than your source's maximum age, the event matched on the board, the market scorable (1X2 and money lines, football handicaps and totals, basketball spreads and totals), and a network bookmaker offering at least your minimum odds.
- **CLV** = best executable price at receipt ÷ no-vig closing price − 1. The close is Pinnacle at kick-off; if Pinnacle publishes none, the last pre-match fair. **Close-beat** = CLV > 0.
- **Recomputed EV** = best executable price ÷ your fair odds − 1. **EV honesty** = Σ recomputed ÷ Σ claimed.
- **Uniqueness** = the selection did not already show an edge of 2 % or more on our board at receipt.
- **Paper P&L** = flat 1 unit at the best executable price, settled on the full-time score of the Pinnacle event; quarter lines split, voids zero. Profit is shown, never qualifies.

## Ladder, sizing, routing

Every new source and every new agent starts here. The full pipeline runs, nothing is placed.

At least 100 proposals over at least 30 days, at least 60 % of them executable. Closing-line value not proven yet.

Closing-line value proven: a close-beat rate of 55 % or more, or an average CLV of +1.5 % or more.

Sustained closing-line value at live_small. A rolling CLV collapse demotes automatically.

**Why closing-line value and not profit:** on 100 to 200 bets, profit is mostly variance; beating the closing line is the only statistically meaningful skill signal at that sample size.

**Sizing.** The operator sets a policy per source: the stake mode (own Kelly on a haircut of the recomputed expected value, a percentage of your stake hint, or fixed), the edge haircut (new sources: only about 30 % of the shown edge is trusted), the odds band, minimum expected value, maximum stake, daily exposure and open bets. You influence sizing with accurate fair prices, honest claims, tight minimum odds, fast submission and a growing closing-line record.

**Routing and attribution.** From live_small on, proposals are executed across the network's partner accounts. Every bet placed from one of your proposals carries your `sourceId` and `proposalId`; the settlement per source is built from those settled bets. Routing weight per tier: shadow 0, qualifying 0, live_small 0.25, live 1, capped by account capacity. Draft terms: 10 % of the net profit of bets executed from your signals, with a high-water mark on your own signal book, paid monthly out of Manometer's share.

## Scorecards and bundles

Your scorecard is private until you publish it: open `manometer.bet/s/<sourceId>`, sign in with your source id and key, and publish; withdraw whenever you like. It shows stake-weighted closing-line value, close-beat rate, executable share, paper yield, EV honesty, uniqueness, drawdown, timing, the ladder with progress per criterion, the cuts by market, lead time, odds band and reason, and every proposal, sealed ones only as receipt and opening round.

The bundle (`/s/<id>/bundle.json`, or *Download my bundle* while private) contains the signed receipts, the time-locked ciphertexts, the opened proposals with their drand beacons, the market row at receipt with its Merkle path, the closing line and the final score with their paths to the hourly anchors, the anchors' OpenTimestamps proofs, the referee's public key and the scoring rules.

## Verifier

`manometer-verify` checks a bundle without trusting our servers. Node 18 or newer; the only dependency is `tlock-js`.

| Command | What it checks |
| --- | --- |
| `receipts` | The Ed25519 signature of every receipt (compare the key with `/netz/pubkey`), the receipt id, the SHA-256 fingerprint of the ciphertext or proposal, the Merkle path to its hourly anchor, the anchor's `.ots` proof and, online, the Bitcoin block header from a public explorer. |
| `seals` | Each ciphertext names its round; it opens with that round's beacon, fetched from drand and not from us, to exactly the published proposal; it was not already open when received; a seal opening before kick-off is marked. |
| `prices` | The market row at receipt belongs to the snapshot in the signed receipt; fair price and best network price are recomputed; the closing line belongs to the same game and is anchored; CLV is recomputed per proposal and on average. |
| `results` | Each result is anchored and belongs to the same Pinnacle event; win, loss, push and half results are recomputed from the full-time score; paper profit and yield are recomputed. |
| `all` | All of the above plus the bundle root: a Merkle root over all entries, so nothing can be removed or added unnoticed. |

```
# Node 18 or newer · the only dependency is tlock-js (drand)
npm install
node manometer-verify.js all nordic-models.bundle.json

# or one part at a time
node manometer-verify.js receipts nordic-models.bundle.json   # signatures, fingerprints, hourly anchors, Bitcoin header
node manometer-verify.js seals    nordic-models.bundle.json   # every seal opens at its drand round, to the published proposal
node manometer-verify.js prices   nordic-models.bundle.json   # market at receipt, closing line, CLV recomputed
node manometer-verify.js results  nordic-models.bundle.json   # results from the final score, paper P&L recomputed

# exit code 0 = everything checked out · 1 = at least one mismatch · 2 = unreadable input
```

`--offline` skips the network lookups and relies on the bundle. Merkle trees: leaf = SHA-256(0x00 ‖ data), node = SHA-256(0x01 ‖ left ‖ right). The verifier's library files are byte-identical copies of the referee's own modules, so it computes exactly what the referee computes.

## Errors and limits

| Code | When |
| --- | --- |
| `200` | `{ ok: true, receipt }`, or the same receipt with `duplicate: true` |
| `400` | Malformed request; a seal that opens in the past or more than 14 days ahead; ids that do not match |
| `401` | Unknown source or wrong key |
| `403` | Source disabled |
| `413` | Too large: 64 KB per request, 32 KB per ciphertext, 16 KB per canonical proposal |
| `429` | 20 failed attempts per address in 10 minutes; 2,000 proposals per source per day; 500 unopened seals per source |
| `503` | Market snapshot not loaded yet, for example right after a restart. Retry in a minute. |

CORS is open (`*`), so a browser-based originator works too.

## Transports

The contract and the receipts are identical on every transport; switching transports changes nothing in your integration logic.

| Transport | For | Status |
| --- | --- | --- |
| **C · HTTPS** | Remote originators. One call per proposal to `/netz/submit`, key in the body. | available |
| **A · Shared folder** | Originators on the same machine as the platform. Append one envelope per line to `feed-inbox.ndjson`; receipts appear in `feed-receipts.ndjson` within about five seconds. Never rewrite or truncate the inbox. | available |
| **B · Extension messaging** | Low-latency messaging between browser extensions, same envelope and receipt. | planned |

## Versioning

This is contract v1. Receipts carry `v: 1`. Additive fields do not change the version; anything that would break an integration gets a new version, announced in advance, with both versions accepted side by side for a transition period.

