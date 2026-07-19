# Fxaurum Rithmic — Storage, caching & calling the API cheaply

How the gateway **stores tick/symbol data on disk**, **caches** derived results, and the **patterns for requesting the API**
so the chart loads fast and the Rithmic feed isn't hammered. Rithmic only serves the **currently-open session** for tick replay
and allows **one instrument search at a time** — both shape the design below.

> Companion docs: **API.md** (message schemas) · **DOM.md** · **HEATMAP.md** · **BIGTRADES.md**.

---

## 1. On-disk stores

All under `%LOCALAPPDATA%\FxaurumRithmic\` (Windows).

| Store | Files | Holds | Partitioned by |
|---|---|---|---|
| **TickStore** (v2) | `TickDb\<EXCH.SYM>\<yyyyMMdd>.tick` + per-symbol `tick.coverage` | every live trade (tick) | **trading session** (1 file = 1 CME session) |
| **Big-trade cache** | `…\big_<tag>_<min>.bigcache` | grouped big-trade bubbles for a **closed** session | trading session |
| **Bar store** | `BarDb\<EXCH.SYM>\<iv>.bars` + per-interval `<iv>.coverage` | klines / tick-derived candles per interval | interval (session-tagged coverage) |
| **Symbol cache** | `TickDb\symbols.json` | instrument search + `instrumentInfo` (tick size, decimals, expiry) | — (whole list, prunes expired) |

### TickStore record format (v3)
- `.tick` — a **32-byte header** (magic `FXTK`, version `3`, recSize `37`) then fixed **37-byte** records: `time` (int64 ms, 8) ·
  `price` (double, 8) · `qty` (int32, 4) · `side` (byte, 1 — `B`/`S`/`0`) · **`tu`** (int64, 8) · **`o`** (int64, 8 — the
  **aggressor's exchange order id**, `0` = unknown). `tu` = exchange match time in
  **microseconds** (`SourceSsboe*1e6 + SourceUsecs`). Group cumulative trades like ATAS by identical
  `tu`; within an event, `o` separates the **first aggressor** from elected stops (bigTrades `a0*` columns).
  **v2 files (29-byte records, no `o`)** stay readable and are appended in their own format; a `Merge` (recanon replay patch)
  always rewrites to v3, and a session file still in v2 triggers a **one-time full-session replay** so `o` gets filled.
  **Live-write dedup** drops only the immediately-repeated
  record matching on **all of `(ms, price, qty, side, tu)`** — that suppresses Rithmic's Image-snapshot resend on (re)subscribe
  **without** dropping two genuine trades at the same ms/price/qty/side (they differ in `tu`), so session volume isn't undercounted.
- `tick.coverage` — one small **text** index per symbol: lines `<tag> <fullUpToMs>
  <flag>` where the flag is `D` = has data · `V` = **verified-complete** (a clean grab reached the session end) · `E` =
  checked-empty · `P` = pre-liquid (skipped) — plus an optional `FINAL <tag>` line meaning the contract is fully captured to
  its last session → never synced again. Flags don't downgrade (rank `V` > `D` > `E`/`P`); `D`/`V` both count as "has data".
  In-RAM cached, atomic rewrite (`File.Move`, not a non-atomic copy). `fullUpTo` = the **replay-verified,
  contiguous-from-session-start** watermark.
  - **Only advanced after a CLEAN replay** (Rithmic completion `RpCode == 0`). A timeout / error / non-zero rp never moves it,
    so a failed grab can't mark a session "done" with missing data.
  - **Self-healing cross-check:** `NeedsSync` doesn't blindly trust `fullUpTo`. If coverage claims a `D` session is complete
    but the file's **real last tick** ends more than the **CME daily-halt window (~90 min)** before the session end, it's a
    lie (the app died mid-session / an old over-claim) → the session is re-synced. So an over-claimed `fullUpTo` self-repairs
    on the next open instead of permanently hiding the tail.
  - **Early-close / holiday is not poison (`V`):** a session that legitimately ends early (e.g. Juneteenth half-day, last
    tick hours before the session end) would trip that 90-min check forever. So when a **clean** grab reaches the session
    end on a closed session, it's marked **`V`** (verified) instead of `D` — and `NeedsSync` trusts `V` outright (skips the
    last-tick check), so a genuine early close is captured once and never re-grabbed. A real missing tail stays `D` (clean
    grab fills it first, *then* `V`), so the self-heal above is unaffected.
  - **Durability:** the merge/rewrite path `fsync`s the temp file then atomically renames; coverage is written **only after** the
    data write succeeds. Killing the process mid-merge leaves the old file intact, never a half-written one.

### Session partitioning (the key idea)
`DayKey(ms) = CentralOf(ms).AddHours(24 − 17).Date` → the **CME Globex trade-date** (session rolls **17:00 CT**, DST-aware via
`TimeZoneInfo`). So **one `.tick` file = one complete session**, never split across UTC midnight. `Read(exch, sym, a, b)`
walks the session files spanning `[a,b]` (skipping the 32-byte header). `TickStore.SessionStartMs(ms)` gives a session's start in
UTC ms. **Sat/Sun** trade-dates are structurally empty (CME closed) → marked `E` and skipped without calling Rithmic.

> **Changing the chart timeframe does NOT touch the tick store.** Ticks are raw; 1m/5m/1h only changes how they're aggregated
> for display. The store is read, never rewritten, on a timeframe change.

---

## 2. Caching layers

- **Big-trade `.bigcache` (per closed session).** `bigTrades` groups raw ticks server-side (by `tu`) into bubbles
  `[t, side, vol, firstPrice, lastPrice]`. For a **closed** session it writes/reads `big_<tag>_<BIG_PERSIST_MIN>.bigcache` (a
  few dozen bubbles → instant). The current session is computed live each time (it's still changing). Cache is valid only if the
  `.bigcache` is **newer than the session's `.tick`** (so a later backfill invalidates it). Client filters further by its own `min`
  (≥ floor). The optional **`merge` (ms)** param (ATAS-style cumulative window) **bypasses this cache** — it's computed fresh so
  the alternate grouping is never persisted under the default key.
- **Bar store (`.bars` + `.coverage`).** Klines are cached per interval; a **closed** session that has stored ticks builds its
  **1m candles from those ticks** (`DeriveMinuteBars`, verified byte-for-byte equal to Rithmic klines), tagged `t` (tick-derived)
  vs `k` (kline) in `<iv>.coverage`. m5/15m/1h/4h aggregate from cached 1m. So reopening a chart / switching timeframe reads
  locally instead of re-hitting Rithmic.
- **Symbol cache (`symbols.json`).** Search is served **cache-first**: a default symbol search returns instantly from the cache;
  a once-per-session background refresh tops it up from Rithmic. Searching `ES` also caches `MES*` (Rithmic "contains ES" returns
  them), so typing `MES` later is a cache hit. Prefix match on symbol (`StartsWith`). Spreads (`-`) and options (space) are
  filtered out and pruned on load. See **API.md `searchSymbol`**.
- **Trade cache (client RAM).** `loadTrades` keeps the live day's trades in memory; footprint/DOM/big-trade seed from it instead
  of each replaying separately. Live trades append to it.
- **Depth cache.** `@depth` is maintained continuously as a `DomBook` and emitted ~30 Hz; clients don't re-request it.

---

## 3. Search is single-in-flight (serialized + retried)

Rithmic allows **one `searchInstrument` at a time**. There is **no API to list all instruments**, and searching by
`InstrumentType` doesn't enumerate. So:

- The gateway **serializes** searches: one active, the rest queued; fast typing **coalesces** to the latest query only.
- On `in use` it **waits 400 ms and retries** the same exchange (≤ 12×) instead of failing.
- The web app picks symbols from a **curated product tree** (expand a product → one targeted `searchSymbol` lists its contract
  months) plus a search box — never a bulk "load everything". This is why there is **no preload-all-on-connect**.

---

## 4. `@depth<N>` — per-subscription depth (bandwidth control)

The live book holds up to **`DOM_MAX_LEVELS` = 1000** levels each side. Each emit is wrapped in a `DepthBox`; `Route` serves each
subscriber its own slice: `@depth<N>` → the **N nearest-to-price** levels, plain `@depth` → all. The web app subscribes
`@depth<N>` with `N` from the Heatmap ⚙ **"Độ sâu sổ lệnh"** (default **100**, shared by the DOM ladder and the heatmap), so you
pull only as deep as you view. Rithmic's MBP (`Quotes`) supplies hundreds of levels — no MBO needed.

---

## 5. Patterns for calling the API cheaply

| Want | Do | Avoid |
|---|---|---|
| Open a chart fast | **Live-first**: subscribe streams + show klines immediately, then **fill history backward** in the background | Awaiting a full-session replay before showing anything |
| Volume profile / footprint history | One `aggTrades` window (cache-first via `loadTrades`); reads the **TickStore** for closed sessions | Re-replaying the same range for footprint, DOM and big-trade separately |
| Big trades for a whole session | Server-side **`bigTrades`** (returns bubbles, not ticks) — uses the `.bigcache` | Pulling millions of ticks to the client and grouping there |
| Order book depth | One `@depth<N>` at the depth you actually display; share it between DOM + heatmap | A separate deep `@depth` per panel |
| Find a symbol | Cache-first `searchSymbol`; let the background refresh top up | A new Rithmic search on every keystroke (serialized → slow + `in use`) |
| Watch many price levels | Pick `N` to match the pane; deep levels rarely change | `@depth1000` at 30 Hz when you only see 100 levels |

**Subscribe order matters:** subscribing during a replay aborts the replay, so the live subscribe runs **before** any background
replay. **Two replay sources:** `replayTrades` = the **current session only** (used for the live tail / footprint of today);
**`replayBars(Tick, 1)`** = the real deep tick history that **does replay CLOSED sessions** — back to roughly the contract's
listing start, **but only while it is still listed** (an expired/delisted contract returns empty). The TickStore is the
long-term archive that accumulates session by session, so once a session is grabbed it survives the contract's delisting.

### Closed-session sync & gap-patch
A closed session is synced by the single-worker queue (`SyncTicksJob`). It is **gap-patch**, not just tail-patch:

- **Has a file →** grab `(fullUpTo → sessionEnd]` via `replayBars(Tick)` and **`Merge`** (window-replace) it in. Because
  `fullUpTo` is the contiguous-verified watermark, every hole **above** it (e.g. from closing/reopening the app several times in
  one session, leaving `■■ gap ■■ gap ■■`) is refilled in one pass — not only the last gap. If `fullUpTo` is poisoned past the
  real data (an old over-claim), the anchor falls back to the session start. A failed (unclean) grab keeps the existing data and
  doesn't advance `fullUpTo`, so it retries next open — **no data loss, no corruption**.
- **No file →** stream the whole session via `TickArchiveStream` (append, low-RAM).
- **Pagination overlap (`TickPager`):** `replayBars(Tick)`'s cursor is **second-granularity** and caps ~10 000 ticks/request. If
  the cap cuts mid-second the old `lastSec+1` advance dropped that second's tail; the pager re-requests from the cut second and
  **dedupes by `(ms,price,qty,tu)`** to recover it (a single second with >cap ticks is the only unrecoverable case, and it
  advances rather than looping).
- **Weekend/maintenance:** `replayBars(Tick)` is rejected with **`rp = 14` (unknown request)** when Rithmic isn't serving tick
  history (time-bar history still works) — surfaced as a yellow `warn` notice; the gap self-fills once the market reopens.

### Archiver scope (opened-symbol-only)
On chart open the webapp `syncArchive` syncs **only the opened contract** (1m+1d bars + ~10 days of ticks, plus a full
last-chance grab if it expires ≤ 15 days). It **does not** fan out to sibling/prior contracts anymore (that caused
ESU6→GCQ6→GCM6 subscribe churn). To capture a prior contract before Rithmic delists it, use the **⤓ Lịch sử** button (on-demand
per contract). A short **linger** (~6 s) holds a Rithmic subscription after its ref-count hits 0, so back-to-back replays reuse
it instead of unsub→resub churn (and that churn no longer races into `in use`/`OMException`).

---

## 6. Build / ops note

`webapp/index.html` and these docs are **embedded resources** compiled into the DLL (and cached in `_resCache`). Editing them on
disk needs a **rebuild + gateway restart** — a browser refresh alone serves the old embedded copy.

---

## 7. Changelog

- **2026-06-21 (later)** — **`V` verified-complete flag + Volume Profile.** Coverage gained a **`V`** flag: a clean grab that
  reaches a closed session's end marks it `V` (vs `D`), and `NeedsSync` trusts `V` outright — so an **early-close / holiday
  half-day** (last tick legitimately hours before session end) is captured once instead of being re-synced every open by the
  90-min cross-check. Flags don't downgrade (`V` > `D` > `E`/`P`). The bar **1m derive** also skips its klines tail-fill when
  the tick session is `V` (the empty tail is real). New **Volume Profile** RPCs read this store — `volumeProfileRange`
  builds N-trading-session profiles (weekends skipped) and self-backfills cold closed sessions; see **VOLUMEPROFILE.md** /
  **API.md**. Startup `rp=11 (no-handle)` removed by gating the first subscribe on `LoginComplete`.
- **2026-06-21** — **Reliability + diagnostics.** Closed-session sync is now **gap-patch** (grab `(fullUpTo→end]` + `Merge`,
  fills mid-session holes from multiple open/close — not just the tail); coverage **only advances on a CLEAN replay**
  (`RpCode == 0`) and **`NeedsSync` cross-checks the real last tick** vs the session end (self-heals an over-claimed `fullUpTo`).
  Live-write **dedup now keys on `tu`** (keeps two genuine same-ms trades). All store writes are **`fsync` + atomic `File.Move`**;
  coverage written only after a durable data write. **`TickPager`** overlap-pagination recovers the tick lost at a cap boundary.
  Subscriptions **linger ~6 s** (kills the sub/unsub churn + the `in use` replay race). Archiver is **opened-symbol-only** (no
  sibling fan-out; ⤓ Lịch sử for prior contracts). Errors now return a **`code`** (OMException `rp` + `omdesc`/`detail`); a global
  first-chance logger + the **`notice` stream** surface meaningful failures to the web (red/yellow/blue banner, click-to-copy),
  filtering out R|API+'s internal login-machinery exceptions. Added a small offline **unit-test** project (`RithmicApi.Tests`).
- **2026-06-18** — **TickStore v2**: one `.tick` file per CME session (32B `FXTK` header + 29B records, **`tu` in-record** — the
  `.fxu` sidecar is gone), a per-symbol `tick.coverage` index (D/E/P + `FINAL`, replaces every `.fxv`), folder `Database`→`TickDb`;
  **auto-migrates** old layouts with zero loss. Big-trade cache `.fxb`→`.bigcache`. Added the **Bar store** (`.bars`+`.coverage`)
  with **1m candles derived from local ticks** (verified == Rithmic klines). `bigTrades` gained **`merge`** (ATAS cumulative
  window; bypasses the cache). Tick sync **skips Sat/Sun** and stops **~10 days before the first liquid session**.
- **2026-06-16** — Session-partitioned TickStore (1 file/session, roll 17:00 CT) + `.fxu` `tu` sidecar; `.fxb` big-trade cache per
  closed session; `@depth<N>` per-subscription depth (shared DOM+heatmap, `DOM_MAX_LEVELS`=1000); serialized + retried search;
  cache-first symbol search with spread/option filtering; live-first chart open with background backfill.
