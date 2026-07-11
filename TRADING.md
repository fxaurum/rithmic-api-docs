# Fxaurum Rithmic — Trading Surface (order entry · SL/TP · chart-trade · panels)

An **ATAS-style order-entry surface** on the web chart: a dock panel to arm an account and fire market/limit/stop orders,
**server-managed SL/TP** (single or multi-level, with breakeven + trailing), **click-to-trade on the chart** with drag-to-set
SL/TP per order, **OCO** pairing, and four session panels — **Accounts · Positions · Trades · Orders**.

The defining idea: **protection lives in the gateway, not the browser.** SL/TP, OCO, breakeven and trailing are reconciled
server-side per `account|symbol`, so **N browser tabs never fight each other** and everything **survives F5 / new tab / gateway
restart**. The client only *shows* state and *sends intents*.

> Companion docs: **API.md** (the raw order/PnL RPCs + report streams) · **RAW.md** (the `@on*report` bus) · **RECIPES.md** · **COPY.md** (mirror one account's trades onto others — reuses these same order/bracket calls).

> ⚠️ **Real money.** Every entry/close is a live order to the exchange. The panel forces a confirm on LIVE accounts until you
> *arm* (arm-once), and every cross-account Close/Cancel asks again. Test on a SIM account first.

---

## 1. Which sockets to call

Order flow is **method + params → gateway → exchange**; results come back on **report streams** (subscribe once). The panels read
**gateway-cached snapshots** via pull RPCs so a fresh tab is instantly correct.

### Subscribe once (per session)

| Stream | Gives |
|---|---|
| `*.*@onpnlupdate` | position + account P&L updates (per-symbol **and** account-summary rows) |
| `*.*@onstatusreport` · `@onmodifyreport` · `@onorderreplay` · `@onopenorderreplay` | order state (new / working / price after a drag / replay image) |
| `*.*@onfillreport` | fills (executions) — drives **Trades** |
| `*.*@oncancelreport` · `@onrejectreport` | cancel / reject |
| `*.*@onfailurereport` · `@onnotcancelledreport` · `@onnotmodifiedreport` | operation failed (RMS / margin / limit) — shown as a banner |
| `*.*@ontraderoute` · `@ontraderoutelist` | trade routes ready (needed before placing) |

### Place / manage orders

| Goal | Call |
|---|---|
| **Market / limit / stop** entry | `placeOrder {account, exchange, symbol, side:"B"\|"S", qty, type:"market"\|"limit"\|"stop", price?, triggerPrice?, duration, tag, tradeRoute?}` |
| **Bracket** (entry + SL/TP tiers) | `bracketOrder {…entry…, targets:[{qty,ticks}], stops:[{qty,ticks}]}` — entry may be limit **or** stop |
| **Modify** (drag a pill) | `modifyOrder {account, orderNum, type, qty?, price?, triggerPrice?}` |
| **Cancel** one | `cancelOrder {account, orderNum}` |
| **Flatten** a position | `flatten {account, exchange, symbol}` (cancels the symbol's protective orders + closes) |
| **Server SL/TP policy** | `setProtection {account, exchange, symbol, enabled, tpOn, slOn, tpOffset, slOffset, tick, duration, pointValue, beOn, beVal, beOff, trOn, trStop, trStep}` |

### Pull snapshots (tab open / F5 / reconnect)

| Call | Returns |
|---|---|
| `getAccounts` | account list (id / fcm / ib / name / RMS) |
| `subscribeOrders` · `subscribePnl` | (re)attach the account's order/PnL plant |
| `replayOpenOrders` · `replayPnl` | ask the exchange to re-send the working-order / position image |
| `positions` | cached position rows (`account\|symbol` → net / avg / open / closed) — **net known immediately**, no ~60 s wait |
| `openOrders` | cached working orders — a new tab rebuilds pills/list without waiting for a live report |
| `accountInfo` | account financials (balance / buying-power / margin / P&L) + static detail (RMS / auto-liquidation) |
| `orderHistory` | session order history incl. Done / Cancelled (State field) |
| `trades` | session fills (executions) |
| `tradeRoutes` · `productRms` | routes + RMS limits (guard entries against the real cap) |

> **Why the caches exist.** The exchange sends each order/position **image only once per subscription**, and the gateway is the
> single subscriber. A tab that connects *after* that image will never receive it — so the gateway keeps the image and serves it
> on `positions` / `openOrders` / `accountInfo` / `orderHistory` / `trades`. On login the gateway also self-runs
> `replayOpenOrders` + `replayPnl` + `replayAllOrders` + `replayExecutions` (last ~24 h) so the panels show the **full session**.

### Multiple connections & account routing

The app can be logged into **several Rithmic accounts at once** (different FCMs/IBs). Every call above carries an `account` (an
`accountId`, or `{fcmId, ibId, accountId}`) and the gateway **routes it to the session that owns that account** — orders, PnL and
pull snapshots all follow the account, not one "primary" login. `getAccounts` returns the accounts of **every** connected session
merged together, so the panel's account dropdown lists them all. (Market-data streams still come from the one **active** login;
switch it with `setDataSource`.)

| Call | Returns |
|---|---|
| `sessions` | one row per login — `{id, name, status, active, accounts:[accountId], rtt, md}`. `name` = the connection's display name (shown instead of the raw account id); `status` ∈ `connected` / `connecting` / `reconnecting` / `disconnected` / `failed`; `active` marks the login currently feeding the chart's data; `accounts` maps each login to its account ids; `rtt` / `md` are the ping / market-data latencies (ms). |
| `setDataSource {id}` | switch which login feeds the chart's market data (the client's ws reconnects). |

The client polls `sessions` to (1) show **connection names** in the account dropdown and the four panels instead of account ids,
(2) **auto-refresh** the dropdown when a new session connects, and (3) **lock the whole panel** when the selected account's login
isn't `connected` (greyed out + input blocked with an "account offline" note; it unlocks automatically on reconnect).

---

## 2. Arming, SIM/LIVE, guards

- **Account dropdown.** The panel trades the account picked in its dropdown, which lists **every account across all connected
  logins** (by connection name). It refreshes as logins connect/drop; if the picked account's login is not connected the panel is
  **locked** until it reconnects.
- **Arm-once.** On a LIVE account the first entry pops a confirm; **Arm** (or confirming once) suppresses further confirms for
  that account this session. **Disarm** re-enables confirms. SIM accounts don't confirm.
- **SIM/LIVE badge.** Auto-classified from the account id; click to flip if wrong.
- **RMS guard.** Before sending, the client checks the account's **BuyLimit / SellLimit / MaxOrderQty** (from `productRms`,
  respecting the `*Flag` — a false flag means "not enforced"). If the account has only margin-based limits, an over-size order is
  rejected by the exchange and shown as a **banner** ("Rejected at RMS — Available margin exhausted"), matching ATAS.

---

## 3. SL/TP protection (gateway-managed)

Toggle **SL/TP** on the panel → the client sends `setProtection` and the **gateway** keeps the position covered. It reconciles per
`account|symbol` on every PnL update: it places the missing legs, resizes qty when you add/reduce (**keeping the dragged price**),
and cancels them when flat. Two modes, chosen in **Edit**:

| Mode | Behaviour |
|---|---|
| **Simple** | One SL + one TP covering the whole net, at `slOffset` / `tpOffset` from the average entry. Drag either pill to move it (SL dragged into profit = a locked-in / breakeven stop, **not** a cancel). |
| **Multi-level · step (ATAS)** | You define N levels `{lot, sl, tp}`. Each Buy/Sell **"takes" the next level** — its lot nets in, its SL/TP become their own bracket lines. After the last level it repeats the last. The panel shows the live legs plus the **▶ next level**. |
| **Multi-level · scale-out (Rithmic)** | One click enters **Σ lot**; the SL/TP fan out into the tiers from that single entry. |

**Breakeven** (move SL to entry ± offset once profit ≥ N) and **trailing** (advance SL by `trStep` for every `trStop` of new
profit, never backward) run **in the gateway** too, from the live P&L.

**Edit dialog.** Editing levels writes a **draft** (the *Disable* button becomes **Save** — saves the config without activating
it); **Apply** switches the panel to that mode but **does not flip the master toggle** — an OFF panel stays OFF and just shows the
config (watch/live only when ON).

---

## 4. Trading on the chart

Turn on **Trading on chart** → the crosshair becomes a price line labelled by what a click would place:

| Position vs market | Left-click (limit) | Right-click (stop) |
|---|---|---|
| Above market | **SELL LIMIT** | **BUY STOP** |
| Below market | **BUY LIMIT** | **SELL STOP** |

- A resting order draws a **pill** (side-coloured: BUY green / SELL red) with **`+TP` / `+SL`** buttons. **Drag `+TP`/`+SL`** onto a
  price → that leg is attached to **that order** (the order is re-placed as a bracket) and auto-added when it fills — **without
  touching the panel config**. A **ghost** dashed line previews the level; **✕** on the ghost removes just that leg.
- **OCO** toggle: the next two chart orders are paired; the first shows **yellow** while it waits; when one **fills or is ✕'d**,
  the gateway cancels the other. The toggle auto-clears after a pair forms.
- **Independent vs position.** A hand-placed order stays labelled `SELL LMT` / `BUY STP` even when it's opposite your position — it
  is **not** relabelled TP/SL and **not** swept by Close (only its own ✕ or **Cancel All** removes it). If it fills, the exchange
  simply nets it into the position (netting model).

---

## 5. Panels (bottom bar)

| Button | Table |
|---|---|
| **Accounts** | one row per account — Balance · Available · Balance-power · Blocked margin · Closed/Open P&L · Commission, plus a per-row **ⓘ → "Full account data"** (Account / Balance / Exposure / Risk-Management / Auto-Liquidation). |
| **Positions** | per `account\|symbol` — Size (FLAT / ±N) · Avg · P&L · Closed P&L · **SL/TP status + config** (`10T / 10T` or `Multiple SL&TP`) · **Close** (confirm). |
| **Trades** | session fills — Account · Instrument · Direction · Volume · Price · **Time (local)**. |
| **Orders** | session orders — **State** (Done / Cancelled / Rejected / Working) · Direction (`Sell Limit`…) · Time · Price (`price/trigger`) · Volume · **Comment** (SL / TP / OCO) · **Cancel** (Working only). |

All four aggregate **every account across every connection** (labelled by **connection name**, not the raw id), update **live**
while open, and seed from the gateway caches on open (so they're complete even for symbols/accounts you never charted). **Trades**
and **Orders** add a **per-account filter** and an independent date **search** (historical fills/orders for a chosen day — it fans
out only to the accounts the filter selects) alongside the live **session** view. Times are shown in the **browser's local
timezone** (converted from the exchange's UTC timestamp).

---

## 6. Multi-tab / reload / restart

| Event | What keeps it correct |
|---|---|
| **N tabs open** | All protection/OCO cancellation is gateway-side (one place). Per-tab ids in order tags stop two tabs forming the same OCO group; a `storage` event syncs config / level / dragged-leg state across tabs. |
| **F5 / new tab** | `positions` → net immediately; `openOrders` → pills/list rebuilt; `accountInfo` / `orderHistory` / `trades` → panels filled — all from the gateway image cache. |
| **Gateway restart / reconnect** | On login the gateway re-images orders + PnL and **adopts** the surviving protective legs (keeps their price, no duplicate set). The client re-subscribes every stream on `onopen` and reseeds. |
| **App closed** | Protective legs are separate resting orders on the exchange, but breakeven/trailing/OCO are gateway logic — so with the app closed those *managed* moves pause; on relaunch the gateway re-adopts and resumes. |

---

## 7. Changelog

- **2026-07-11** — Cross-referenced **COPY.md** — the CopyEngine mirrors one account's trades onto others using these same
  order/bracket calls.
- **2026-07-10** — **Multi-account routing.** The app now runs **several Rithmic logins at once**: every order / PnL / pull RPC
  routes by `account` to the owning session, `getAccounts` merges all logins, and a new `sessions` RPC exposes each login's
  connection name + status + accounts (+ `setDataSource` to pick the market-data source). The panel's **account dropdown** lists
  all connected accounts (auto-refreshing as logins connect/drop); the four panels show **connection names** instead of ids and
  let **Trades / Orders filter by account** with an independent date search; and the panel **locks** whenever its selected
  account's login isn't connected.
- **2026-07-03** — Bottom-bar panels: **Accounts** (+ Full-account-data ⓘ), **Positions** (Close), **Trades**, **Orders**
  (Cancel), with full-session history via `replayExecutions` / `replayAllOrders` on login; `accountInfo` / `positions` /
  `openOrders` / `orderHistory` / `trades` pull RPCs backed by gateway image caches. Clean order Comment (SL/TP/OCO).
- **Trading surface** — order-entry dock (arm-once, market/ASK/BID), gateway-managed **SL/TP** (single + multi-level step/FOCCA)
  with **breakeven + trailing**, **chart-trade** (click limit/stop, per-order drag-to-set SL/TP + ghost), **OCO** pairing; N-tab /
  F5 / restart hardening (image caches + leg adoption + reconnect reseed).
