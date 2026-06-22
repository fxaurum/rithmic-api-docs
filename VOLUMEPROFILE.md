# Fxaurum Rithmic — Volume Profile (volume-at-price)

A **volume-at-price** histogram overlay: horizontal bars showing **how much volume traded at each price**, plus the
**VPOC** (point of control — highest-volume price), **Value Area** (VAH/VAL), and per-price coloring. Drawn as an LWC
primitive at the **left/right edge** (composite) or **anchored to each session's candles** (per-session). Three modes
cover the current session (native Rithmic) and any number of closed sessions (built from the local tick store).

> Companion docs: **API.md** (`volumeProfile`, `volumeProfileRange` RPCs) · **FOOTPRINT.md** (same tick source) ·
> **STORAGE.md** (the TickStore the range modes read + how closed sessions self-fill) · **RECIPES.md** (§4, the generic
> build-it-yourself algorithm) · **DOM.md** (the DOM has its own single-day session profile).

---

## 1. Which sockets to call

Volume Profile is **request/response (RPC)**, not a stream — you ask once and re-poll. There are **two** RPCs; the webapp
picks one by mode:

| Goal | Call | Returns |
|---|---|---|
| **Current session, native** (no local data needed) | `volumeProfile {exchange, symbol}` | `{rows:[[price,vol]], total, vpoc, rp}` — Rithmic's own `getVolumeAtPrice` for the **live session only**. `rp` 0=ok · 7=no-data · -1=timeout. |
| **N closed sessions, composite** (one merged profile) | `volumeProfileRange {exchange, symbol, startTime, endTime, days}` | `{rows, total, vpoc, sessions:<count>}` — merged volume-at-price across the sessions. |
| **N closed sessions, one profile each** | `volumeProfileRange {exchange, symbol, startTime, endTime, days, split:"1"}` | `{sessions:[{start, end, rows, total, vpoc}], total, split:true}` — one profile per session, each carrying its time band (`start`/`end` ms) so the client can anchor it under that session's candles. |

- **`days`** = number of **trading sessions** to include (the gateway walks back from the current session **skipping
  weekends** via `IsWeekendSession`, so 6 days that straddle Sat+Sun still yields 6 profiles, not 4). `startTime`/`endTime`
  are the fallback window when `days` is absent.
- The range modes read the **local TickStore** for each session. A **closed** session whose store is empty is
  **backfilled once** (`replayBars(Tick,1)` → merged → next call reads local). The **current** session is read live (no
  backfill). See **STORAGE.md**.
- `volumeProfileRange` bins by the **instrument tick**; the **"Gộp tick"** setting re-bins coarser **client-side** (so it
  applies to all three modes uniformly, no re-fetch of the server bins).

**Minimum viable VP** = one `volumeProfile` call (current session, native — works even with zero local history).

---

## 2. The three modes (⚙ → "Chế độ")

| Mode | What it draws | Source |
|---|---|---|
| **Phiên hiện tại (daily)** | One profile for the **live session**, pinned to the chart edge. | Native `getVolumeAtPrice` (`volumeProfile`). |
| **Gộp N ngày (comp)** | **One merged** profile over the last **N trading sessions**, pinned to the chart edge. | `volumeProfileRange` (merged) from the TickStore. |
| **Mỗi phiên (sess)** | **One profile per session**, each anchored to that session's candle band (slides with the chart). | `volumeProfileRange split=1`. |

- **Edge-pinned** (daily, comp): the histogram grows from the left (or right — see "Bên") edge of the pane; bar length ∝
  volume, scaled to the profile's **own global max** (so bars **don't resize** when you zoom/pan price in and out of view).
- **Per-session** (sess): each session's histogram fills that session's **full candle width** (open→close), anchored by
  **bar index** (not wall-clock — the chart compresses weekend/missing-bar gaps, so a wall-clock map would drift). It
  **slides with the candles** when you pan, and is skipped when its band is fully off-screen.

---

## 3. Settings (gear ⚙ next to the VP toggle)

| Setting | Meaning | Default |
|---|---|---|
| Chế độ (mode) | daily / comp / sess (above) | daily |
| Số ngày (days) | Number of **trading sessions** (comp + sess; weekends skipped) | 5 |
| Bề rộng histogram (width) | Max bar length in **px** (edge-pinned modes) | 160 |
| Gộp tick (bin) | Merge **N ticks per price bucket** — coarser profile, fewer rows (1 = native tick) | 1 |
| Bên (side) | Edge-pinned histogram grows from **Trái (left)** or **Phải (right, next to the price axis)** | left |
| Value Area (%) | Value-Area target as **% of total volume** (VAH/VAL span out from VPOC to this) | 70 |
| Line VPOC | Draw the VPOC as a full-width yellow price line | on |
| Line VAH/VAL | Draw VAH/VAL as dashed blue price lines | on |

The profile auto-refreshes every **30 s** (the live session keeps growing). Changing **days / mode / Gộp tick** re-fetches;
changing **VA% / width / side** recomputes or redraws in place (no RPC).

---

## 4. VPOC & Value Area

- **VPOC** (Volume Point of Control) = the price row with the **largest** volume. Drawn yellow.
- **Value Area** = the price band around the VPOC holding **VA%** (default 70%) of total volume. Computed by expanding
  out from the VPOC one row at a time, each step taking the **adjacent side with the larger** volume, until the
  accumulated volume reaches the target. **VAH** = top of that band, **VAL** = bottom.
- In **sess** mode VPOC/VA are computed **per session**; in comp/daily they're computed on the single merged profile. VA%
  changes recompute in place (no re-fetch).

---

## 5. Rendering & internals

- **Edge-pinned (comp/daily)** — `panel = min(width, 45% of pane)`; each row `len = vol / globalMax × panel`, drawn from
  the chosen edge at the row's price coordinate. Row thickness is derived from two adjacent price coordinates (so it
  scales with vertical zoom). Scaling to **global max** (not the visible-max) keeps bars stable while panning price.
- **Per-session (sess)** — for each session, the first/last **bar index** in `[start,end)` is found by time (binary
  search of `bars[]`); x is `timeToCoordinate` on-screen and `anchor + Δindex × barSpacing` off-screen. `drawW =
  bandWidth × 0.92`; bars grow from the session's open. This is the same **map-by-bar-index** rule the footprint /
  big-trade renderers use.
- **Gộp tick** — merges rows into coarser buckets (N ticks per bucket) client-side, then VPOC/VA/max are recomputed on the
  binned rows.
- **Self-filling history** — in comp/sess a closed session with an empty store triggers a one-time **tick backfill**
  (paged `replayBars(Tick,1)`) merged into the store; the next call serves it from local. The current session is never
  backfilled (live ticks fill it). See **STORAGE.md** for the coverage flags (`D`/`V`/`E`/`P`).

---

## 6. Native vs. local — when each is used

| | `volumeProfile` (native) | `volumeProfileRange` (local) |
|---|---|---|
| Sessions | current (live) only | any N closed (+ current) |
| Source | Rithmic `getVolumeAtPrice` | local TickStore (self-backfilled) |
| Needs local history | no | yes (auto-fills closed sessions) |
| Bin size | Rithmic's | instrument tick, then "Gộp tick" client-side |
| Cost | 1 call, instant | 1 call; first time on a cold session also runs a tick backfill |

---

## 7. Changelog

- **2026-06-21** — Volume Profile: composite (`comp`) + per-session (`sess`) modes over the local TickStore alongside the
  native current-session (`daily`) mode; `days` = **trading sessions** (weekends skipped); each session read **from its
  open** (oldest no longer half-drawn); **per-session anchored by bar index** (slides with candles, no edge-stick, no
  squish on pan); composite bars scaled to **global max** (no resize on zoom/pan); **"Gộp tick"** setting (N tick/bucket);
  settings ⚙ (mode / days / width / bin / side / VA% / VPOC+VA lines).
