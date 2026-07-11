# Fxaurum Rithmic — Copy Trade (CopyEngine)

Mirror the fills of one **leader** account onto one or more **follower** accounts — **across FCMs/IBs** — from a
single app. Modelled on **R Trader Pro's *Trade Copier*** (Copy From → Copy To, Qty Multiplier, Fractional Qty
Rounding, Convert-to-Market) but implemented **in the gateway** so it also works when the leader and follower sit
on **different clearing firms**, which Rithmic's own copier cannot do.

> Companion docs: **TRADING.md** (order entry / gateway-managed SL/TP — the same order + bracket calls the copier
> reuses) · **API.md** (the WebSocket envelope) · **RAW.md §3 ④** (`placeOrder` / `bracketOrder` params).

> ⚠ **Real money.** Every mirrored fill is a live order on the follower account. The engine ships **off by
> default**, and each pair is created **Disabled** — nothing copies until you turn on both the engine and the
> pair. **Test on SIM accounts first.**

---

## 1. Why it lives in the gateway (not on Rithmic's server)

R Trader Pro's Trade Copier is **server-side**: it sends `omd_set_order_copy_leader_account` /
`omd_set_order_copy_follower_account` to the Omne backend and **Rithmic mirrors the orders for you** — but only
**within one login (same FCM/IB)**. It cannot copy from an account at FCM *A* to an account at FCM *B*.

Because the multi-connection app logs into **several Rithmic accounts at once** (see **TRADING.md** →
multi-connection), it can watch the leader's fills on one session and place the mirror on the follower's session
itself. So the CopyEngine is a **client-side mirror** running in the gateway process:

- **Leader fill in → follower order out.** The engine subscribes to the leader account's `@onfillreport` (already
  available per-account) and, on each fill, places the matching order on the follower via that follower's own
  session (`FeedFor(follower).placeOrder`).
- **Any FCM to any FCM.** Futures symbols are globally unique (`ESU6` is `ESU6` at every CME clearing firm), so no
  symbol mapping is needed.
- **Config persists** to `%LOCALAPPDATA%\FxaurumRithmic\copytrade.json` and reloads on start.

The UI model (hierarchical Leader → Follower grid, Qty Multiplier, Fractional Qty Rounding, Convert-to-Market
seconds) and the field semantics are deliberately kept identical to R Trader Pro so the behaviour is familiar.

---

## 2. Pairs & clusters

A **pair** is one `leader → follower` link with its own settings. All pairs that share a leader form a **cluster**
(shown as one collapsible group with the leader's connection name and a **Master** badge; each follower row shows a
**Follower** badge). One leader can fan out to many followers; a follower may also be a leader in another cluster
(chains are allowed, and the loop-guard below stops them running away).

| Concept | Meaning |
|---|---|
| **Leader (Copy From)** | The account whose fills / working orders are the source. |
| **Follower (Copy To)** | The account that receives the mirror. |
| **Cluster** | Every pair with the same leader — enable/disable or delete as a group. |
| **Enable (per pair)** | The pair copies only when **this** toggle **and** the engine are on. |
| **Copy Engine (kill-switch)** | Master on/off for the whole engine. Off ⇒ nothing copies, regardless of pairs. |

---

## 3. Sizing — multiplier, %-equity, rounding

The follower quantity is the leader fill size times an **effective multiplier**, then rounded:

- **Sizing = × (fixed).** `followerQty = round(leaderQty × mul)`. Use a whole number (2 = double, 0.5 = half).
- **Sizing = %eq (equity-scaled).** `effMul = mul × (followerBalance / leaderBalance)`, taken from each account's
  cached `AccountBalance`. The follower auto-scales to its own account size; the `mul` box is an extra factor
  (usually `1.00`). If a balance is missing, it **falls back to fixed** `mul`.
- **Rounding.** `Round up` or `Round down` decides how a fractional result (e.g. `1 × 0.5 = 0.5`) becomes an
  integer lot. Round down can yield `0` (skipped); round up guarantees ≥ 1.

---

## 4. Two copy modes

Each pair copies in exactly one of two ways, decided by the **Copy SL/TP** toggle.

### 4a. Market-only mirror (Copy SL/TP **off**) — default

Every leader fill (open, add, reduce, close) is mirrored as a **market** order on the follower. The follower simply
tracks the leader position fill-for-fill: when the leader closes, the closing fill mirrors straight through and
closes the follower. No SL/TP is placed — the follower's protection **is** the leader's exit. This is the simplest
and most robust mode; the **reconcile** loop (§7) keeps it aligned if a fill is ever missed.

### 4b. Follower's own bracket (Copy SL/TP **on**), multi-tier

When on, an **opening** leader fill makes the follower enter with its **own FOCCA bracket** — the SL/TP **tick
distances are copied from the leader's bracket** and the follower then exits on that bracket, independent of the
leader. This only works when the leader placed its SL/TP **through this app** (a `bracketOrder`); an
externally-placed (e.g. ATAS) leader has no bracket to copy, so those pairs behave as market-only.

- **Multi-tier.** If the leader's bracket has several TP and/or SL tiers (scale-out), **all** tiers are copied. The
  follower quantity is split across the tiers **proportionally to the leader's tier quantities**, the last tier
  absorbing the rounding remainder so the tiers sum to exactly the follower entry size.
  *Example:* leader enters 3 lots with TP `2@10t + 1@20t`; follower at `× 2` enters 6 lots with TP `4@10t + 2@20t`.
- **Debounce.** Opening fills are accumulated for ~400 ms and flushed as **one** bracket, so an entry that fills in
  several pieces doesn't create several brackets.
- Closing/flattening leader fills are **suppressed** in this mode — the follower's own bracket handles the exit
  (avoids a double close).

---

## 5. Fast fill (pre-fire) — per follower

Normally the follower reacts to the leader's **fill** (one round-trip: leader places → Rithmic fills → gateway
sees the fill → follower places). **Fast fill** removes that wait for orders the app itself sends:

- When the leader places a **MARKET** order **through this app**, the follower's market order is fired
  **immediately, in parallel** (using the order quantity, not waiting for the leader fill).
- The leader order's tag is marked so, when its fill later arrives, it is **not mirrored again** (no double).
- Fast fill applies only to **market** orders that originate **in the app**, and only on **market-only** pairs
  (Copy SL/TP off). A leader order placed on an external platform (ATAS) does not pass through here, so it falls
  back to normal mirror-on-fill.
- **Risk:** if the leader's market order is *rejected* after the follower already pre-fired, the follower is
  briefly out of sync — the **reconcile** loop (§7) detects and unwinds the excess within ~20 s. Tag markers expire
  after 30 s.

---

## 6. Order-mirror for working orders + Convert-to-Market (opt-in)

By default only fills copy. Turn on **Mirror lệnh chờ / Mirror pending orders** (a toolbar toggle) to also mirror
the leader's **working limit / stop orders** on market-only pairs:

- Leader places a working **limit/stop** → the follower gets a copied working order (same price/trigger, size ×
  effMul). Leader **modifies** → follower modifies. Leader **cancels** → follower cancels.
- **Convert-to-Market (seconds).** Per follower: when the leader's limit **fills** but the follower's copied limit
  has **not** filled within *N* seconds, the follower's order is cancelled and re-sent as **market** so it doesn't
  miss the move. Blank = never · `0` = immediately · `N` = after N seconds. (This is R Trader Pro's
  "Convert to Market After Leader's Order is Filled".)
- When a leader fill was already handled by order-mirror, its market mirror is skipped (no double).

> Limitations: order-mirror is market-only pairs; externally-placed OCO legs without a recognised tag may be
> mirrored; a modify that arrives before the follower order id is learned is dropped.

---

## 7. Reconcile (drift correction)

A 20-second timer (only while the engine is on) checks every **market-only** pair for drift: it compares the
follower's net position against `round(leaderNet × effMul)` using the **PnL cache** (the authoritative position
image). If they differ by ≥ 1 lot — a missed fill, a network blip, a rounding edge, a fast-fill reject — it places
a **market** order to close the gap and logs a `reconcile` entry. A 12-second guard skips any symbol that was just
mirrored, so it never races a mirror that's still settling. Copy-SL/TP pairs are excluded (their follower runs its
own bracket, so a deliberate divergence there is expected).

---

## 8. Safety model

| Guard | Effect |
|---|---|
| **Engine off by default** | `engineOn = false` until you flip the Copy Engine switch. |
| **New pair Disabled** | A freshly-added pair does nothing until you enable it. |
| **Loop guard** | Every order the engine places is tagged `cpy…`; the engine ignores fills with that tag, so a follower's own fill can't re-trigger a copy (safe even in leader→follower→leader chains). |
| **Fill dedup** | Leader fills are de-duplicated by `ReportId`. |
| **Reconcile** | Bounds drift on market-only pairs (§7). |
| **SIM badge** | Accounts are classed SIM/LIVE from their id; test on SIM first. |
| **Disable all** | One toolbar action disables every pair without deleting the config. |

---

## 9. Which sockets to call

All copy control is a small set of RPCs on the same `ws://127.0.0.1:9001/` gateway. Each **returns the full config**
(so the UI re-renders from one reply); the engine does its work by itself from the leader's report streams — you do
**not** subscribe to anything special to copy.

| Goal | Call |
|---|---|
| Read current config | `copyConfig {}` |
| Master kill-switch | `copyEngine {on: true|false}` |
| Enable order-mirror for working orders | `copyMirror {on: true|false}` |
| Add / update a pair | `copyUpsert {leader, follower, mul?, roundUp?, enabled?, sizeMode?, mktAfter?, copySltp?, fastFill?}` |
| Remove one pair | `copyRemovePair {leader, follower}` |
| Remove a whole cluster | `copyRemoveCluster {leader}` |
| Enable / disable a cluster | `copyClusterEnable {leader, enabled}` |
| Disable every pair | `copyDisableAll {}` |
| Read the activity log | `copyLog {}` |

**Config reply** (`copyConfig` and every mutating call):

```json
{
  "engineOn": false,
  "mirrorPending": false,
  "pairs": [
    {
      "leader": "<leaderAccountId>", "leaderName": "Prop A",
      "follower": "<followerAccountId>", "followerName": "Prop B",
      "mul": 2.0, "roundUp": false, "enabled": false,
      "sizeMode": 0, "mktAfter": -1,
      "copySltp": false, "fastFill": false, "exitMaster": true
    }
  ]
}
```

| Field | Meaning |
|---|---|
| `mul` | Quantity multiplier (double). |
| `roundUp` | `true` = round fractional lots up, `false` = down. |
| `enabled` | Per-pair on/off (needs the engine on too). |
| `sizeMode` | `0` = fixed `mul` · `1` = %-equity (`mul × followerBal/leaderBal`). |
| `mktAfter` | Convert-to-market seconds: `-1` never · `0` immediately · `N` after N s (order-mirror only). |
| `copySltp` | `true` = follower runs its own copied bracket (§4b); `false` = market-only mirror (§4a). |
| `fastFill` | `true` = pre-fire app-originated market orders (§5). |
| `exitMaster` | Legacy field, retained for config compatibility; **unused**. |

**Activity log reply** (`copyLog`, newest first, ~250 entries):

```json
{ "log": [
  { "t": 1720000000000, "type": "mirror", "leader": "…", "follower": "…",
    "sym": "ESU6", "side": "B", "qty": 2, "note": "x2" }
] }
```

`type` is one of `mirror` · `fastfill` · `bracket` · `reconcile` · `error`; `t` is Unix ms.

---

## 10. R Trader Pro parity

| R Trader Pro | Here | Note |
|---|---|---|
| Copy From / Copy To grid | ✅ | hierarchical Leader → Follower cluster |
| Quantity Multiplier (`57126`) | ✅ | `mul` |
| Fractional Qty Rounding (`24000`) | ✅ | `roundUp` |
| Convert to Market — seconds (`57128`) | ✅ | `mktAfter` (order-mirror mode) |
| Enable / Disable per row + Enable/Disable all | ✅ | `enabled` · `copyClusterEnable` · `copyDisableAll` |
| Same-FCM server-side copy | ↔ | done **client-side** instead so it also spans **different FCMs** |
| %-equity sizing | ➕ | app extra (`sizeMode = 1`) — R Trader Pro is multiplier-only |
| Copy SL/TP (multi-tier) | ➕ | app extra — follower runs a copied bracket |
| Fast fill (pre-fire) | ➕ | app extra |
| Kill-switch + reconcile + activity log | ➕ | app extras |

---

## 11. Limitations

- **Copy SL/TP** only copies a bracket the leader placed **through this app**; an external (ATAS) leader falls back
  to market-only.
- **Order-mirror / Convert-to-Market** is market-only pairs; untagged external OCO legs may be mirrored; a modify
  before the follower order id is learned is dropped.
- **Fast fill** covers only app-originated **market** orders; a reject-after-pre-fire is corrected by reconcile
  (~20 s), not instantly.
- Position **flips** (long→short within one fill) aren't specially handled beyond fill-for-fill mirroring +
  reconcile.
- Requires the follower account to be **connected** (and its trade routes primed) — an offline follower's copy
  waits for reconnect + reconcile.

---

## 12. Roadmap

1. **Copy SL/TP for external leaders** — synthesise a follower bracket from the leader's own protective working
   orders when there's no app `bracketOrder`.
2. **Per-symbol filters / max size** — allow-list symbols and cap follower size.
3. **Follower guard** — auto-flatten + disable a follower on repeated errors (Replikanto-style).
4. **Explicit flip handling** — detect and mirror a same-fill reversal as a distinct close+open.

---

## 13. Changelog

- **2026-07-11** — **Multi-tier SL/TP + Fast fill + add-form fields.** Copy SL/TP now copies **all** of the
  leader's bracket tiers (proportional split, remainder on the last tier) instead of only the first. **Fast fill**
  (per follower) pre-fires app-originated market orders without waiting for the leader fill, tag-marked so the fill
  isn't double-mirrored, with reconcile covering a reject-after-pre-fire. The add-pair form gained Sizing / Convert
  / Fast fill so a new pair is fully configurable up front.
- **2026-07-11** — **Phase 2.** Drift **reconcile** (20 s, market-only), an **activity log** (`copyLog`, colour-coded
  panel), **%-equity sizing** (`sizeMode`), and opt-in **order-mirror** of working limit/stop orders with
  **Convert-to-Market** (`mirrorPending` + `mktAfter`).
- **2026-07-10** — **CopyEngine MVP.** Gateway-side cross-FCM mirror-on-fill (`copyConfig` / `copyEngine` /
  `copyUpsert` / `copyRemovePair` / `copyRemoveCluster` / `copyClusterEnable` / `copyDisableAll`), kill-switch,
  per-pair enable, Qty Multiplier + Fractional Qty Rounding, connection-name UI, and **Copy SL/TP** (follower's own
  bracket, single tier). Engine off by default; new pairs Disabled.
