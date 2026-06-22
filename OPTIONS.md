# Fxaurum Rithmic — Options chain · GEX · 0DTE

Two linked features built on the **options chain** of the futures contract you're viewing:

1. **Options popup** — the live CALL · Strike · PUT grid (bid/ask/last/vol/OI per strike, per expiry).
2. **GEX** (Gamma Exposure) — dealer gamma per strike, drawn on the chart as **Gamma-flip / Call Wall / Put Wall / Spot** price lines + a fixed **profile panel**; aggregated across expiries in a **DTE window** (incl. **0DTE**).

Both pull the same chain via Rithmic **`getInstrumentByUnderlying`** (the future is the underlying), so they're spot-consistent and need the **options-data entitlement**.

> Companion docs: **API.md** (`@bookTicker`, `@trade`, `instrumentByUnderlying`) · **STORAGE.md** · **RECIPES.md**.

---

## 1. Which sockets to call

GEX/options need **no history replay** — chain discovery + a one-off OI/quote sweep, then live recompute.

| Goal | Call | Why |
|---|---|---|
| **Discover expiries** of the future | `instrumentByUnderlying {underlying, exchange, expiration:""}` | 1 call → list of option **expiry dates (CCYYMMDD)** for that future (weekly/daily/monthly, auto-correct underlying). |
| **Strikes of one expiry** | `instrumentByUnderlying {…, expiration:<CCYYMMDD>}` | drill a date → the option instruments (symbol, strike, C/P, product). |
| **OI + bid/ask per strike** | `@bookTicker` (stream) | the only source of **open interest** (`oi`) + best bid/ask; GEX **batch-subscribes** all strikes ±band, snapshots, then unsubscribes. |
| **Last** (popup) | `@bookTicker` (`l`) + `@trade` | `@bookTicker.l` carries the last trade **incl. the snapshot Rithmic sends on subscribe**, so an illiquid strike shows a Last immediately (no waiting for a live print); `@trade` updates it live. |
| **Volume / Net per strike** (GEX weighting) | `@bookTicker` (`v` / `nv`) | session **gross** volume (`v`) and **net** = buy−sell by aggressor (`nv`) — for the Volume/Net GEX weighting modes; both live. The gateway pushes `v`/`nv` immediately for chain-covered options (doesn't wait for a quote). |
| **Spot** (GEX) | the future's `@trade`/`@bookTicker` (already subscribed by the chart) | `F` = live future price; GEX recomputes against it every 2.5 s. |

**Always filter out Event Contracts** (`/^EC/` — hourly event options like `ECES`/`ECG*`): they are not standard options on the future, and gold EC report a **10× scaled strike**.

> **`subscribeByUnderlying` (wired via `chainSub`):** the OI sweep and the popup now bulk-subscribe a whole expiry's chain in **one call** (`chainSub` → Rithmic `subscribeByUnderlying`) instead of one `subscribe` per strike — the gateway skips the per-strike Rithmic subscribe for covered symbols; quotes/OI/last still route via each strike's `@bookTicker`. Far fewer subscribe calls (dodges the per-strike storm / "in use" throttle). Trade-off: Rithmic streams every strike of the expiry to the gateway. See **API.md → `chainSub`**.

---

## 2. Options popup (CALL · Strike · PUT)

- **One IBU head call** lists **all expiry dates** of the future, capped at the **future's own expiration** (so e.g. ESU6 shows dates up to its Sep expiry, not other futures' options). The dropdown lists every date as `dd-mm-yyyy (Nd)`.
- **Lazy drill**: only the **selected** expiry is drilled (strikes fetched) + subscribed (`@bookTicker`/`@trade`). Switching expiry fetches that one on demand (already-seen ones are cached). Opening stays fast even with 30+ expiries.
- **Empty (Event-Contract-only) dates auto-prune**: if a selected date drills to no standard strikes, it's removed from the dropdown and the next valid date is shown.
- **Viewport subscribe**: the strike grid renders full, but only the ~20 rows on screen are subscribed (IntersectionObserver) → light.
- **ATM row + ITM shading follow live price** (yellow ATM strike; CALL ITM = strike < spot, PUT ITM = strike > spot), updated every 500 ms without rebuilding the table.

---

## 3. How GEX is computed

**Per strike `K`, summed across every expiry in the DTE window:**

```
GEX(K) = Σ_expiries [ Γ(K,exp)·W_call(K,exp) − Γ(K,exp)·W_put(K,exp) ]  × mult × F² × 0.01
```

…where **`W` is the positioning weight** — choose in ⚙ (see *Weighting* below): **OI** (default), **Volume**, or **Net**.

- **Black-76** (options on futures), client-side. **IV is solved from the option mid** (Newton + bisection) because the feed gives no greeks.
- **One gamma per strike, from the OTM-side IV**: Black-76 gamma is identical for call & put at a strike, so use the *reliable* IV — put IV when `K ≤ spot`, call IV when `K > spot` — clamped to **[2%, 300%]**. (Solving IV from a deep-ITM price ≈ intrinsic returns garbage σ → gamma explodes → **false walls** at deep-ITM strikes. The OTM IV + clamp kills this.)
- **Each expiry uses its own time-to-expiry `T`** (hours-aware, floored ~2 h) so **0DTE gamma is large but not infinite**, while far expiries get small gamma. ⇒ the sum at each strike is **naturally dominated by the near expiries** (high gamma); far quarterly OI adds little despite large OI.
- **Sign = SqueezeMetrics/SpotGamma convention**: dealers **long calls, short puts** → call gamma **+**, put gamma **−**. This is an *assumption* (flipping it is one line).
- **Multiplier** (`GEX_MULT`, ES=50…) only scales the **$ magnitude** — flip/wall **locations are independent of it**. `r = 4%` hardcoded.

### Weighting (⚙) — what counts as "positioning"

| Mode | `W` = | Realtime? | When |
|---|---|---|---|
| **OI** (default) | Open Interest | ❌ **EOD** — CME publishes OI once each morning; **static intraday for ALL options incl. 0DTE** | multi-DTE / swing; the structural surface |
| **Volume** | session **gross** volume (all fills, both sides) | ✅ live (`@bookTicker.v`) | **0DTE / intraday** — where today's activity concentrates |
| **Net** | session **net** = buy − sell by **aggressor** (signed) | ✅ live (`@bookTicker.nv`) | flow direction (closer to positioning) — best on **liquid 0DTE (ES)** |

- **OI lags intraday** — today's 0DTE positions don't hit OI until tomorrow (expired by then). So for same-day flow, Volume/Net are the live proxies; OI is the EOD structure.
- **Net is signed + small + noisy** — it swings/flips per trade (on thin chains like gold a single fill flips a strike), so the walls/flip (which need a clear +/− net) can flicker or vanish. Net shines on **liquid** 0DTE; on thin chains prefer **Volume** or **OI**.
- **Net seed:** the live `nv` only counts from when GEX turns on; the full-session net is **seeded once per session** from `aggTrades` (today's prints, buy−sell), cached client-side in `localStorage["gexnetseed:<EXCH.SYM>"]` so re-toggling within the session **doesn't re-hit Rithmic** (options aren't in the tick store → replay is the only source). `net = seed + (nv_live − baseline)`.

**Work split:**
- **OI mode** — static intraday → **sweep once** (cache `gexcache:`), **recompute every 2.5 s** from cache + live `F` (no feed calls). **Re-sweep OI every 50 s**; **re-enumerate** the chain every **15 min** (self-rolls across days/sessions/expiries).
- **Volume / Net mode** — keep the chain **subscribed live** (via `chainSub`), so `v`/`nv` update realtime; **no periodic re-sweep** (the band is fixed until you re-toggle GEX). Net replays the seed only for strikes not yet in the session cache.

---

## 4. Key levels

| Level | Definition | Trading read |
|---|---|---|
| **Call Wall** | strike with the **largest positive net GEX** (calls dominate) | resistance / upside magnet; price often stalls there |
| **Put Wall** | strike with the **largest negative net GEX** (puts dominate) | support / downside magnet |
| **Gamma flip** (γ-flip / zero gamma) | the spot price where **total net GEX = 0** | **above** → dealers long gamma → vol dampened, price pinned/mean-revert; **below** → dealers short gamma → moves amplified, trendy/squeeze-prone |
| **Top-N lines** | the N largest **+** (green) and N largest **−** (red) net-GEX strikes | secondary magnets/levels |

**Call Wall / Put Wall use NET GEX** (not raw max call/put OI) — a strike with huge call OI *and* huge put OI is net-neutral, not a wall. The Top-N lines **skip** the strikes already drawn as walls (no duplicate lines).

---

## 5. The profile panel

- A **fixed-width panel pinned to the right edge** (gap-from-axis + width are settings), with a **centre line**: **GEX+ → right/green**, **GEX− → left/red** (net per strike). Or **Call-Put mode**: Call → right/green, Put → left/red separately.
- **Scaled to the strikes currently in the visible price range** (the largest *on-screen* bar uses the full panel width) — so the profile is always readable whatever the zoom; a giant strike scrolled off-screen no longer shrinks everything to nothing.
- **GEX does NOT rescale the chart** (no auto-fit / autoscaleInfoProvider) — Y-axis drag stays free; turning GEX off leaves the scale untouched.
- **Top-left legend** (bold, font size configurable): a **DTE tag**, net GEX, Call GEX (green) / Put GEX (red), then flip / Call Wall / Put Wall.
  - DTE tag: `[0DTE]` (yellow = today's expiry in use) or `[1d…Xd×N]` (gray) = nearest…farthest DTE × number of expiries aggregated.

---

## 6. 0DTE & DTE modes

GEX aggregates a **DTE window** — which expiries to include is the key choice:

| Mode (⚙) | Includes | Use for |
|---|---|---|
| **DTE window = 0** | only **today's** expiry (pure **0DTE**) | same-day pinning; the sharpest intraday signal |
| **DTE window = 2–3** (recommended for scalping) | 0DTE + next 2–3 days | set-and-forget; always populated even when there's no same-day expiry |
| **DTE window = N** | all expiries ≤ N days | wider context |
| **"All to contract expiration"** (checkbox) | every expiry up to the **future's own expiry** (e.g. ESU6 → 18-Sep) | the **total** gamma surface; re-sweeps every **10 min** (heavier). Per-symbol. |

**Why near-term matters most:** gamma ∝ 1/√T, so a 0DTE ATM strike has *enormous* gamma and a 90-day strike *tiny* gamma. The window-0/small modes isolate the gamma actually driving today's hedging; "all to expiration" is valid too (far expiries self-attenuate via low gamma) but blends in the structural picture. **Don't** use "only the terminal/quarterly expiry" for intraday — it has the lowest gamma and misses the near-term pin.

**Session roll:** at the new session (~17:00 CT) the re-enumeration picks up the new nearest expiry automatically. If there's no same-day (0DTE) expiry, GEX falls back to the nearest upcoming one and the tag shows gray `[1d]` instead of yellow `[0DTE]` (walls/flip levels are still valid at 1–2 DTE; only the same-day pin effect is what's missing).

---

## 7. Settings (gear ⚙ next to the GEX toggle) — all **per-symbol**

| Setting | Meaning | Default |
|---|---|---|
| Cách trục giá (gap) | profile panel gap from the price axis, px | 10 |
| Bề rộng (width, split +/−) | total panel width, px (each side = half) | 150 |
| Cửa sổ DTE | aggregate expiries ≤ this many days (**0 = pure 0DTE**) | 7 |
| Lấy tất cả đến đáo hạn HĐ | include every expiry up to the future's expiry (ignores the window; 10-min update) | off |
| Trọng số theo VOLUME | weight by session **gross volume** (live) instead of OI — for 0DTE/intraday | off (OI) |
| └ dùng NET (buy−sell) | sub-option of Volume: weight by **net** = buy−sell aggressor (signed, live, session-seeded) | off |
| 🗑 Xoá cache & tính lại | clear all GEX caches (`gexcache:` OI + `gexnetseed:` net seed, every symbol) and re-sweep from scratch (fresh Rithmic) | — |
| Cỡ chữ label | legend font size, px | 14 |
| Top N line GEX | draw N largest +(green) / −(red) net strikes as lines | 0 |
| Profile tách Call/Put | Call→right / Put→left instead of Net | off |
| Line Call Wall / Put Wall / Gamma flip / Spot | toggle each price line | on |

Each symbol remembers its own GEX settings (and footprint/heatmap/DOM/timeframe) via per-symbol profiles; opening a symbol restores them.

---

## 8. Assumptions & caveats (read before trading on it)

- **Real data**: strikes, OI, expiries, bid/ask come straight from Rithmic.
- **Model (interpretation, not fact)**: the dealer **long-call/short-put sign** convention, **IV from a mid snapshot**, **r = 4%**, and the multiplier (scales $ magnitude only). Flip/wall/profile are **positioning interpretation**.
- **Strike-scale gotcha**: Event Contracts report a **10× strike** (e.g. `44650` for a 4465 strike) — filtered out (`/^EC/`), but confirm once visually that kept strikes ≈ spot before trading.
- **Carry**: aggregating expiries that settle into the *same* future (e.g. all ES options up to the Sep quarterly → ESU6) is spot-consistent. A very wide window that crosses a futures roll mixes spot references — prefer **front future + a window within its block**.
- **OI is end-of-day**, even for 0DTE — it does not move intraday. Use **Volume/Net** weighting for live same-day flow. **Net** is signed/noisy (esp. on thin chains like gold → walls flicker/vanish); it's most meaningful on **liquid 0DTE (ES)**. The net **sign** reuses the OI dealer convention — if the flow read looks inverted, flip it (one line).

---

## 9. Changelog

- **2026-06-19** — **GEX weighting modes (OI / Volume / Net)** + bulk-subscribe. `W` in the GEX sum is now selectable: **OI** (EOD, default), **Volume** (live gross, `@bookTicker.v`), **Net** (live buy−sell, `@bookTicker.nv`). Volume/Net keep the chain **subscribed live** via **`chainSub`** (Rithmic `subscribeByUnderlying`, 1 call/expiry — dodges the per-strike subscribe storm), no periodic re-sweep. Net is **seeded once per session** from `aggTrades` (cached `gexnetseed:`, reused across toggles → no re-hit Rithmic). Gateway pushes `v`/`nv` immediately for chain-covered options + on `@bookticker` subscribe (no waiting for a quote). Added a **🗑 Clear cache** button (wipes `gexcache:`+`gexnetseed:` and re-sweeps). `@bookTicker` gained `l` (last incl. subscribe snapshot) + `nv` (net flow). See **API.md → `chainSub` / `@mbo`**.
- **2026-06-17** — GEX/options: chain via `getInstrumentByUnderlying` (1 call + lazy drill, EC filtered); per-strike aggregation across a DTE window with hours-aware `T`; Black-76 IV from mid + OTM-side gamma (deep-ITM false-wall fix); Call/Put Wall = net-GEX extremes, gamma flip, Top-N lines; fixed right-edge profile (Net or Call-Put) scaled to **visible** strikes; "all to contract expiration" mode (10-min); options popup lists **all** expiries to the contract expiry (lazy, empty/EC dates pruned); 0DTE tag `[0DTE]`/`[1d…Xd×N]`; all GEX settings **per-symbol**; self-roll across sessions.
