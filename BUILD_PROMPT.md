# Build prompt — dựng lại web dashboard từ API này

> Copy toàn bộ phần dưới, paste vào một AI khác (Claude / GPT / …). Nó sẽ đọc tài liệu API
> (file `.md` gộp ở trang chủ, hoặc các trang `api.html`/`raw.html`/`OPTIONS.md`…) rồi dựng lại
> một dashboard "Market Structure" (GEX + orderflow) chạy trên Fxaurum Rithmic Gateway.
>
> Tech stack đã cố định; phần đặt lệnh (order-entry) tạm bỏ qua.

---

# NHIỆM VỤ: Dựng web dashboard "Market Structure" (GEX + orderflow) cho futures/options

Bạn là kỹ sư full-stack. Dựng web trading dashboard kiểu "Turtle Hub – Market Structure"
(dark theme, nhiều panel), lấy DỮ LIỆU THẬT từ **Fxaurum Rithmic Gateway** — WebSocket API
kiểu Binance phát data futures/options Rithmic (R|API+).

## TECH STACK (CỐ ĐỊNH — dùng đúng bộ này)
- **Vite + React 18 + TypeScript** (SPA, mỗi panel là 1 component).
- **TradingView Lightweight Charts** (`lightweight-charts`) cho nến + volume; các overlay tuỳ
  chỉnh (GEX profile, footprint, net-dollar, heatmap) vẽ bằng `<canvas>` riêng chồng lên chart
  hoặc custom series/primitive của thư viện.
- **Zustand** làm store realtime (1 store nhận data từ WS; mỗi panel subscribe slice của nó).
- **Tailwind CSS** cho dark dashboard.
- **WebSocket native** — viết 1 lớp `RithmicClient` theo mẫu TypeScript trong `examples.html`
  (map `id`→Promise cho RPC; `stream`→handler cho push; tự reconnect).
- **Black-76 greeks thuần TS** (gamma/vanna/charm + normal CDF tự code) — không cần thư viện options.
- Không backend riêng (gateway chính là backend). Deps tối thiểu.

## BƯỚC 1 — ĐỌC TÀI LIỆU TRƯỚC KHI CODE
- Docs online: https://fxaurum.github.io/rithmic-api-docs/ (trang chủ có nút "⬇ Tải toàn bộ tài liệu"
  → tải `fxaurum-rithmic-api-docs.md` gộp MỌI .md — đọc hết).
- Trọng tâm: `api.html`/`API.md` (API tổng hợp) · `raw.html`/`RAW.md` (raw R|API+) ·
  `examples.html` (client 9 ngôn ngữ — lấy mẫu TS) · `OPTIONS.md` (GEX/0DTE) · `FOOTPRINT.md` ·
  `VOLUMEPROFILE.md` · `DOM.md` · `BIGTRADES.md`. Mục `#rawmap` cho biết mỗi API tổng hợp gọi raw nào.

## KẾT NỐI
- `ws://127.0.0.1:9001/` — JSON 1 object/message, timestamp Unix ms, symbol UPPERCASE, không auth.
- SUBSCRIBE stream `{method:"SUBSCRIBE",params:["CME.NQU6@trade"],id}` → `{stream,data}` liên tục.
- RPC `{method,params,id}` → `{id,result}`.

## PANEL → API
| Panel | Gọi API |
|---|---|
| Nến giá + khung 15m… | `klines` (lịch sử) + `@kline_<iv>` / tự gom `@trade` cho nến live |
| GEX PROFILE + STRIKE PROFILE (STRIKE/$GEX/C-OI/P-OI) | `instrumentByUnderlying` (2 bước: ngày→strike) hoặc `optionChain`; `chainSub` sub cả chuỗi; OI + bid/ask mỗi strike từ `@bookTicker` (field `oi`). Net GEX / 0DTE / Call-Put GEX / FLIP / MAX PAIN / ATM IV / 1σ / VANNA / CHARM tính client-side bằng Black-76 (xem OPTIONS.md) |
| BIG TAPE (lệnh lớn MUA/BÁN theo $) | `bigTrades` (lịch sử) + `@trade` (live), lọc ≥ ngưỡng $ |
| NET DOLLAR (buy/sell theo giá) | footprint từ `aggTrades` (`[t,p,q,side,o,tu]`) + `@trade` → gom buy/sell$ theo giá |
| Volume Profile / VPOC / Value Area | `volumeProfile` (phiên) · `volumeProfileRange` (nhiều phiên) |
| THANH KHOẢN / OB / DOM ladder | `@depth` / `@depth<N>` |
| Heatmap / Liq Zones | `@depth` hoặc `@mbo` (DBO order-by-order) |
| CVD | cộng dồn delta từ `@trade` (AggressorSide) |
| Tick size / point value / hết hạn | `instrumentInfo` |
| Ô chọn symbol | `searchSymbol` |
| Level Behavior (Respect/Absorb/Fakeout/Break) | tự tính: giá vs mức GEX (call/put wall, flip, HVL) + `@trade` |

> Các chỉ số FUNDING / RETAIL L/S / COT INDEX / nguồn "binance" KHÔNG đến từ gateway này (sentiment ngoài) — bỏ qua.

## RÀNG BUỘC
- Gateway local-only `ws://127.0.0.1:9001`, không expose.
- Tự reconnect: gateway tự khôi phục subscription, client KHÔNG cần SUBSCRIBE lại.
- Stream tần suất cao (`@trade`/`@bookTicker`/`@depth`) nặng — chỉ sub panel đang hiển thị, UNSUBSCRIBE khi ẩn.

## OUTPUT
- SPA layout: thanh trên (symbol/khung/tab) · chart trái · GEX profile giữa · strike profile +
  level behavior + net dollar phải · big tape dưới · dark theme.
- Panel lịch sử (GEX/VP/big tape) load 1 lần rồi cập nhật live; panel live cập nhật realtime theo stream.
- Bắt đầu 1 symbol (vd `CME.NQU6`), đổi qua `searchSymbol`.

Trước khi code: đọc file `.md` gộp, tóm tắt kiến trúc + danh sách stream/RPC sẽ dùng, rồi mới dựng.

## PHẢN HỒI (quan trọng)
Trong lúc đọc API và dựng app, nếu bạn phát hiện **vấn đề** (tài liệu thiếu/mâu thuẫn, API khó
dùng, thiếu field cần thiết, payload không đủ để dựng panel, bug, rủi ro) HOẶC có **đề xuất cải
tiến** (thêm RPC/stream/field, đổi format, tối ưu luồng dữ liệu, gom call…) → **ĐỪNG tự ý sửa
gateway**. Hãy **liệt kê rõ ràng để người dùng xem và quyết định**: mỗi mục ghi (1) vấn đề/đề
xuất, (2) ảnh hưởng tới việc dựng panel nào, (3) gợi ý cách giải quyết. Cứ dựng phần làm được
trước, đánh dấu chỗ phải workaround vì API chưa hỗ trợ.

> NOTE FOR THE AI: while reading the API / building, if you hit any **problem** (missing or
> contradictory docs, awkward API, missing field, insufficient payload, bug, risk) or have an
> **improvement idea** (new RPC/stream/field, format change, data-flow optimization) — do **not**
> change the gateway yourself; **list it clearly for the user to review and decide** (problem →
> which panel it affects → suggested fix). Build what's possible first and flag any workarounds.
