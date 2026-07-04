# Raw R|API+ passthrough — every R|API+ callback as a socket

> Companion docs: **API.md** (the curated, higher-level streams/RPCs) · **MBO.md** · **DOM.md** · **examples.html** (client code, 9 languages)

This exposes **every** Rithmic R|API+ callback over the WebSocket, **1-to-1**:

| R\|API+ surface | socket shape |
|---|---|
| **push callback** (data arrives continuously) | a **stream** `<exch>.<sym>@<CallbackName>` |
| push callback **without a symbol** (account / order / exchange list / user…) | a **global stream** `*.*@<CallbackName>` |
| **pull / query method** (you ask, one answer comes back) | an **RPC** call |

The curated streams (`@trade`, `@depth`, `@mbo`, `@kline`, `@bookticker`, `@mboraw`…) are transforms built **on top** of these — use those for charting. Use the raw bus when you want the untouched Rithmic event with all its fields.

---

## 1. Stream form (push → stream)

Subscribe `<exch>.<sym>@<CallbackName>` (case-insensitive). The callback name is exactly the R|API+ callback, e.g.:

```
SUBSCRIBE  cme.esu6@onsettlementprice
SUBSCRIBE  cme.esu6@onmarketmode
SUBSCRIBE  cme.esu6@onopeninterest
```

Each message is the Info struct reflected to JSON — `e` = callback name, then **all** its public fields:

```json
{ "stream":"cme.esu6@onsettlementprice",
  "data":{ "e":"OnSettlementPrice", "Exchange":"CME", "Symbol":"ESU6",
           "SettlementPrice":7611.50, "SettlementDate":"20260626", "Ssboe":1718000000, … } }
```

**Global (no symbol)** events use the reserved base `*.*`:

```
SUBSCRIBE  *.*@onaccountupdate
SUBSCRIBE  *.*@onpnlupdate
SUBSCRIBE  *.*@onfillreport
```

### What turns the feed on
- A **symbol** raw stream calls `Subscribe(All)` for that instrument → you immediately get every L1 / price / settlement / limit / market-mode / OI / volume / indicator / reference callback.
- **Book / depth-by-order / bar** callbacks (`OnLimitOrderBook`, `OnBidQuote`, `OnAskQuote`, `OnDbo`, `OnBar`…) need their own feed — subscribe the curated stream that activates it **alongside** the raw stream: `@depth` (book), `@mbo`/`@mboraw` (DBO), `@kline` (bars). The raw bus then passes their events through too.
- **Global** streams flow automatically after login — no activation needed.
- **Query-response** callbacks (`OnOrderReplay`, `OnExecutionReplay`, `OnOptionList`, `OnRefData`…) do **not** self-push — a method must trigger them; the raw bus calls it for you on subscribe (see the `trigOf` map in `raw.html`). This is R|API+'s request/response pair: you call `REngine.replayX()`, Rithmic fires back the `XReplay` callback — so **the method name ≠ the callback name**. Most line up (`replayOpenOrders`→`OnOpenOrderReplay`, `replayExecutions`→`OnExecutionReplay`) but some don't — `replayAllOrders()` → `OnOrderReplay` (method has "All", callback doesn't), so it can't be derived by a string rule and must be looked up. The `On`-prefix + lowercasing (`onorderreplay`) is this gateway's stream naming; raw R|API+ names the callback `OrderReplay` (no "On").

### Cost
The passthrough only runs while a client holds that event (ref-counted); when nobody subscribes a callback, zero work. High-frequency callbacks (`OnQuote`, `OnBidQuote`, `OnAskQuote`) are heavy — only subscribe what you consume.

---

## 2. Stream catalog (by cluster)

Every callback below is a stream. `S` = symbol-scoped (`exch.sym@…`), `G` = global (`*.*@…`).

### Connection / session — G
`OnAlert` · `OnLoginCompleted` · `OnEnvironment` · `OnEnvironmentList` · `OnPing`

### L1 quotes — S
`OnBidQuote` · `OnAskQuote` · `OnBestBidQuote` · `OnBestAskQuote` · `OnBestBidAskQuote` · `OnQuote` · `OnQuoteReport` · `OnEndQuote`

### Trades — S
`OnTradePrint` (each print, ns timestamp + aggressor) · `OnTradeCondition` · `OnTradeVolume` (running session volume) · `OnAggregator`

### Session prices / OHLC — S
`OnOpenPrice` · `OnHighPrice` · `OnLowPrice` · `OnClosePrice` · `OnHighBidPrice` · `OnLowAskPrice` · `OnMidPrice` · `OnCloseMidPrice`

### Settlement — S
`OnSettlementPrice` · `OnProjectedSettlementPrice`

### Limits & market state — S
`OnHighPriceLimit` · `OnLowPriceLimit` (limit up/down) · `OnMarketMode` (open/closed/halt/auction) · `OnOpeningIndicator` · `OnClosingIndicator`

### Open interest — S
`OnOpenInterest`

### Book / depth-by-order — S *(feed via `@depth` / `@mbo`)*
`OnLimitOrderBook` · `OnDbo` · `OnDboBookRebuild`

### Bars — S *(feed via `@kline`)*
`OnBar` · `OnBarReplay`

### Volume profile — S
`OnVolumeAtPrice` (native `getVolumeAtPrice`, `Prices[]`/`Volumes[]`)

### Reference / instrument — S
`OnRefData` · `OnAuxRefData` · `OnPriceIncrUpdate` (tick size / decimals)

### Instruments / options / strategy — G *(usually request/response — see §3)*
`OnExchangeList` · `OnInstrumentSearch` · `OnInstrumentByUnderlying` · `OnOptionList` · `OnStrategy` · `OnStrategyList` · `OnEquityOptionStrategyList` · `OnBinaryContractList` · `OnUserDefinedSpreadCreate` · `OnEasyToBorrow` · `OnEasyToBorrowList` · `OnTradeRoute` · `OnTradeRouteList`

### Account / position / PnL / RMS — G
`OnAccountList` · `OnAccountUpdate` · `OnPnlUpdate` · `OnPnlReplay` · `OnPositionExit` · `OnProductRmsList` · `OnAutoLiquidate` · `OnSodUpdate`

### Orders — replay + reports — G
`OnOrderReplay` · `OnOpenOrderReplay` · `OnSingleOrderReplay` · `OnExecutionReplay` · `OnOrderHistoryDates` · `OnLineUpdate` · `OnBracketReplay` · `OnBracketUpdate` · `OnBracketTierModify` · `OnFillReport` · `OnStatusReport` · `OnModifyReport` · `OnCancelReport` · `OnBustReport` · `OnFailureReport` · `OnRejectReport` · `OnNotCancelledReport` · `OnNotModifiedReport` · `OnTradeCorrectReport` · `OnTriggerReport` · `OnTriggerPulledReport` · `OnOtherReport`

### User / IB / agreement — G
`OnUser` · `OnUserList` · `OnUserProfile` · `OnAssignedUserList` · `OnIbList` · `OnAgreementList` · `OnPasswordChange`

> ⚠ Account / order / PnL / RMS / user streams require the corresponding Rithmic **trading entitlement** and expose account data over the socket. Subscribe them only in trusted/local setups.

---

## 3. Call form (pull → RPC)

RPC names = the **REngine method** (the old Binance-style names still work as **aliases** in the dispatch). Send `{ "method":"<name>", "params":{…}, "id":N }`. Two sub-kinds:

- **② Query (inline)** — you call → **one `result` comes back inline** (`{id,result}`; error `{id,error}`).
- **③ Trigger** — you call → an **ack**, and the data arrives on a **stream `@On*`** (subscribe it FIRST). These are the request side of query-response callbacks.

### ② Query — inline result

| RPC (REngine) | params | → result (callback shape) | alias cũ |
|---|---|---|---|
| `listExchanges` | — | `OnExchangeList` | `exchangeInfo` |
| `getAccounts` | — | `OnAccountList` (cache) | `account` |
| `instrumentInfo` | `{exchange,symbol}` | **gộp** `OnRefData`+`OnPriceIncrUpdate` (object phẳng) | — (composite) |
| `searchInstrument` | `{query,exchange}` | `OnInstrumentSearch` | `searchSymbol` |
| `getOptionList` | `{exchange,product,expiration}` | `OnOptionList` | `optionChain` |
| `getInstrumentByUnderlying` | `{underlying,exchange,expiration?}` | `OnInstrumentByUnderlying` | `instrumentByUnderlying` |
| `getVolumeAtPrice` | `{exchange,symbol}` | `OnVolumeAtPrice` | `volumeProfile` |
| `getAuxRefData` | `{exchange,symbol}` | `OnAuxRefData` | `auxRefData` |
| `getProductRmsInfo` | `{accountId?}` | `OnProductRmsList` | `productRms` |
| `getStrategyList` | `{exchange,product,…}` | `OnStrategyList` | `strategyList` |
| `getStrategyInfo` | `{exchange,symbol}` | `OnStrategy` | `strategyInfo` |
| `getEquityOptionStrategyList` | `{exchange,underlying,…}` | `OnEquityOptionStrategyList` | `equityOptionStrategies` |
| `listBinaryContracts` | `{exchange,product}` | `OnBinaryContractList` | `binaryContracts` |
| `getEasyToBorrowList` | — | `OnEasyToBorrowList` | `easyToBorrow` |
| `listTradeRoutes` | — | `OnTradeRouteList` | `tradeRoutes` |
| `listOrderHistoryDates` | `{accountId?}` | `OnOrderHistoryDates` | `orderHistoryDates` |
| `getUserProfile` | — | `OnUserProfile` ⚠️PII | `userProfile` |
| `listIbs` · `listUsers` | — · `{ibId,userType}` | `OnIbList` · `OnUserList` | — |
| `listAgreements` | `{accepted?}` (def `true`) | `OnAgreementList` | — |
| `listAssignedUsers` | `{accountId?}` | `OnAssignedUserList` | `assignedUsers` |
| `listEnvironments` · `getEnvironment` | — · `{key}` | `OnEnvironmentList` · `OnEnvironment` | — |

Each fires the query and returns once (8 s timeout → `null`). Reply = same reflected-Info JSON as the streams.

### ③ Trigger — ack + data on stream `@On*`

| RPC (REngine) | params | → data stream |
|---|---|---|
| `replayPnl` | `{accountId?}` | `@OnPnlReplay` |
| `replayOpenOrders` | `{accountId?}` | `@OnOpenOrderReplay` |
| `replayAllOrders` | `{accountId?,startTime,endTime}` | `@OnOrderReplay` |
| `replayHistoricalOrders` | `{accountId?,date}` | `@OnOrderReplay` |
| `replayExecutions` | `{accountId?,startTime,endTime}` | `@OnExecutionReplay` |
| `replaySingleOrder` | `{accountId?,orderNum}` | `@OnSingleOrderReplay` |
| `replaySingleHistoricalOrder` | `{accountId?,orderNum,date}` | `@OnSingleOrderReplay` |
| `replayBrackets` | `{accountId?}` | `@OnBracketReplay` |
| `replayQuotes` | `{accountId?}` | `@OnQuoteReplay` |
| `subscribePnl` | `{accountId?}` | `@OnPnlUpdate` (auto lúc login) |
| `subscribeOrders` | `{accountId?}` | `@On*Report` (auto lúc login) |
| `subscribeAutoLiquidate` | `{accountId?}` | `@OnAutoLiquidate` |

> `instrumentInfo` giữ tên (gateway **gộp** 2 method REngine). Gateway-only / Binance-style (KHÔNG phải raw R|API+): `klines`, `aggTrades`, `bigTrades`, `volumeProfileRange`, `continuousInfo`, `syncHistory`, `syncTicks`, `tickInfo`, `syncStatus` — xem API.md. Một số stream/RPC ít dùng (`@onuserprofile`…/`listIbs`…) **vẫn chạy** nhưng đã ẩn khỏi bảng tương tác trong `raw.html` cho gọn — vẫn liệt kê đầy đủ ở trên.

### ④ Order entry — write (TIỀN THẬT, composite → REngine)

> ⚠ **Real money.** Composite RPCs build the R|API+ params from friendly JSON and call raw `REngine` **write** methods. Reply = an **accept ack** (`{ok:true, accepted:true}`); the real outcome (working/filled/rejected) arrives on the report streams (`@OnStatusReport`/`@OnFillReport`/`@OnRejectReport`/`@OnBracketUpdate`), keyed by `ExchOrdId`/`OrderNum`. Gateway is local-only (`ws://127.0.0.1:9001`); a **trading-entitled** account is required. No Execute buttons for these in `raw.html`.

**Lifecycle:** ① `subscribeOrders` once per account (turns on order+bracket updates **and** loads trade routes — needed to place orders) → ② subscribe the report streams → ③ send `placeOrder` / `modifyOrder` / `cancelOrder` / `exitPosition` / `bracketOrder` → ④ reply = accept-ack; outcome on the report streams. **Account** resolved: `{fcmId,ibId,accountId}` → `accountId` (in logged-in accounts) → first active. **Trade route** auto-picked from cache by exchange (prefer Default+UP); override with `tradeRoute`. No cached route for the exchange → `placeOrder` returns `409`.

| RPC (composite) | → REngine (raw) | params |
|---|---|---|
| `placeOrder` | `sendOrder` (builds `*OrderParams`) | `side` · `type`(market/limit/stop/stoplimit) · `qty` · `price` · `triggerPrice` · `duration` · `tradeRoute?` · `tag?` |
| `modifyOrder` | `modifyOrder` | `orderNum` · `type` · `qty?` · `price?` · `triggerPrice?` |
| `cancelOrder` | `cancelOrder` | `orderNum` |
| `cancelAllOrders` | `cancelAllOrders` | `accountId?` |
| `exitPosition` / `flatten` | `exitPosition` | `exchange` · `symbol` (flatten market) |
| `bracketOrder` | `sendBracketOrder` (builds `BracketParams`) | entry + `targets[]` / `stops[]` (ticks) — OCO |
| `subscribeOrders` | `subscribeOrder` + `subscribeBracket` | `accountId?` — once/account |
| `tradeRoutes` (= `listTradeRoutes`) | `listTradeRoutes` | — (order route per exchange) |

**`placeOrder`** — `type` selects the param struct built from your JSON:
```json
{ "method":"placeOrder", "id":2, "params":{
    "accountId":"DEMO123", "exchange":"CME", "symbol":"ESU6",
    "side":"B", "type":"limit", "qty":1,
    "price":7611.50, "triggerPrice":7610.00,
    "duration":"DAY", "tradeRoute":"globex", "tag":"my-strat" } }
```
| `type` | builds | needs |
|---|---|---|
| `market` | `MarketOrderParams` | — |
| `limit` | `LimitOrderParams` | `price` |
| `stop` / `stopmarket` | `StopMarketOrderParams` | `triggerPrice` |
| `stoplimit` | `StopLimitOrderParams` | `price` + `triggerPrice` |

**`bracketOrder`** — entry + target(s)/stop(s) (OCO); `ticks` = offset from entry. `bracketType` auto: both → *target and stop*, stops only → *stop only*, targets only → *target only*.
```json
{ "method":"bracketOrder", "id":7, "params":{
    "accountId":"DEMO123", "exchange":"CME", "symbol":"ESU6",
    "side":"B", "type":"market", "qty":2, "price":7611.50,
    "targets":[ {"qty":1,"ticks":8}, {"qty":1,"ticks":16} ],
    "stops":[   {"qty":2,"ticks":10} ] } }
```

**modify / cancel / exit** (`qty`/`price`/`triggerPrice` applied only when present):
```json
{ "method":"modifyOrder",    "id":3, "params":{ "accountId":"DEMO123","exchange":"CME","symbol":"ESU6","orderNum":"8246531","type":"limit","qty":2,"price":7611.75 } }
{ "method":"cancelOrder",    "id":4, "params":{ "accountId":"DEMO123","orderNum":"8246531" } }
{ "method":"cancelAllOrders","id":5, "params":{ "accountId":"DEMO123" } }
{ "method":"exitPosition",   "id":6, "params":{ "accountId":"DEMO123","exchange":"CME","symbol":"ESU6" } }
```

**Value mappings (friendly → Rithmic):**

| field | accepts | Rithmic |
|---|---|---|
| `side` | `B`/`BUY`, `S`/`SELL`, `SS`/`SELLSHORT` | `B` · `S` · `SS` |
| `duration` | `DAY`, `GTC`, `FOK`, `IOC` | `DAY` · `GTC` · `FOK` · `OC` |
| `entryType` | `M`/`MANUAL`, `A`/`AUTO` | `M` · `A` (default manual) |
| `type` (modify) | `market`/`limit`/`stop`/`stoplimit` | `M` · `L` · `STP` · `SLMT` |

**Not exposed (raw write — `raw.html` NOEXP):** batch/niche — `sendOrderList`, `sendOcaList`, `cancelOrderList`, `modifyOrderList`, `linkOrders`, `modifyBracketTier`, `modifyOrderRefData` (param-collection complexity); `createUserDefinedSpread`; `changePassword` (security → native dialog).

---

## 4. Notes

- Field names/types are exactly as the R|API+ SDK exposes them (no renaming). Enums are stringified; arrays are passed through (capped at 512).
- A callback you don't subscribe costs nothing. Unsubscribe to stop both the passthrough and (for symbol streams) the underlying market-data ref.
- This is additive — it doesn't change any curated stream.

## 5. Confirmed payload shapes

Real shapes captured live (via the raw.html **📋 Copy** button). Growing as callbacks are verified.

### Field glossary (chung cho nhiều callback)

| Field | Ý nghĩa |
|---|---|
| `e` | tên callback (gateway thêm vào, vd `"OnBidQuote"`) |
| `CallbackType` | **enum**: `Image` (snapshot lúc (re)subscribe) · `Update` (cập nhật live) · `History` (từ replay lịch sử) · `Undefined` |
| `UpdateType` | **enum** (quote/book): `Solo` (1 mức đơn) · `Begin`/`Middle`/`End` (1 batch nhiều mức: đầu/giữa/cuối) · `Clear` (xoá mức) · `Aggregated` (gộp) · `Undefined` |
| `Exchange` / `Symbol` | sàn / mã |
| `Price` / `Size` | giá / khối lượng tại mức đó |
| `NumOrders` | số lệnh tại mức giá |
| `ImpliedSize` | size suy ra từ spread/strategy (implied liquidity) |
| `LeanPrice` | giá tham chiếu nội bộ Rithmic (xấp xỉ fair/mid) |
| `Ssboe` / `Usecs` | mốc thời gian SÀN: giây-từ-epoch + micro-giây. ms = `Ssboe*1000 + Usecs/1000` |
| `SourceSsboe`/`SourceUsecs`/`SourceNsecs` | mốc tại NGUỒN (sàn): giây / micro / nano-giây |
| `JopSsboe`/`JopNsecs` | mốc tại Rithmic **JOP** (join-order-plant): giây / nano-giây |
| `AggressorSide` | `B`/`S` — bên chủ động (taker) là mua/bán |
| `AggressorExchOrdId` / `ExchOrdId` | mã lệnh **aggressor** / mã lệnh **khớp** (resting) của sàn |
| `Condition` | cờ điều kiện trade (rỗng = trade thường) |
| `Date` (settlement) | ngày settlement, định dạng `CCYYMMDD` |
| `PriceType` (settlement) | `final` (chính thức) · `preliminary` (sơ bộ) |
| `MarketMode` (OnMarketMode) | trạng thái phiên: `Open` · `Closed` · `Pre-Open` · `Pause` · `Halt` · `Auction` … |
| `Event` / `Reason` (OnMarketMode) | sự kiện (`No Event`…) / lý do trạng thái (`Group Schedule` = theo lịch nhóm) |
| `VolumeFlag` / `Volume` | có khối lượng hợp lệ / khối lượng (vd OnOpeningIndicator) |
| `QuantityFlag` / `Quantity` (OnOpenInterest) | open interest hợp lệ / số hợp đồng đang mở |
| `Side` | `B` = bid/mua · `S` = ask/bán |
| `Priority` (OnDbo) | thứ tự ưu tiên FIFO của lệnh trong hàng tại mức giá |
| `SizeFlag` (OnDbo) | size của lệnh hợp lệ |
| `PreviousPrice` (OnDbo) | giá cũ khi lệnh đổi giá (NaN nếu không đổi) |
| `RpCode` | mã kết quả Rithmic, `0` = OK (book/VP/refdata) |
| `Vwap` / `NumTrades` (OnBar) | VWAP phiên / số lệnh khớp trong nến |
| `BuyerAsAggressorVolume` / `SellerAsAggressorVolume` | KL mua / bán chủ động (delta footprint) |
| `Prices[]` / `Volumes[]` (OnVolumeAtPrice) | hai mảng **song song** (cùng index) = volume profile theo giá |
| `SinglePointValue` (OnRefData) | giá trị 1 điểm (vd $20/điểm NQ) |
| `Expiration` / `ExpirationTime` | hết hạn `CCYYMMDD` / epoch-giây |
| `SettlementMethod` / `UnitOfMeasure` (OnAuxRefData) | `cash`/`physical` / đơn vị đo hợp đồng |
| `Rows[]` (OnPriceIncrUpdate) | bảng bước giá: `PriceIncr`(tick) · `Precision`(số lẻ) · `FirstPrice`/`LastPrice`(khoảng áp dụng, 0/0=mọi giá) · `First`/`LastOperator` |
| `ProductCode` / `InstrumentType` | mã sản phẩm (`NQ`) / loại (`Future`/`Future Strategy`/`Future Option`) |
| `PutCallIndicator` / `StrikePrice` / `Underlying` (option) | `Call`/`Put` · giá thực hiện · mã cơ sở |
| `StrategyType` / `Legs` (strategy) | loại spread (`Iron Condor`…) / các chân của spread |
| `FcmId` / `IbId` / `TradeRoute` / `Status` | định danh FCM · IB · tên route · `UP`/`DOWN` (route đặt lệnh) |
| `AccountId` / `AccountName` | mã / tên tài khoản (PnL/order) — *ví dụ trong doc đã ẩn* |
| **state-wrapper** `{Ignore,Clear,Use,State,Value}` | field PnL/account: đọc `Value` chỉ khi `State:"Use"` (`Clear`→null · `Ignore`→bỏ qua) |
| `AccountBalance`/`OpenPnl`/`ClosedPnl`/`Position` | số dư · lãi/lỗ mở · lãi/lỗ chốt · vị thế ròng (đều state-wrapper) |
| `RmsInfo` / `Algorithm` (RMS) | cấu hình rủi ro tài khoản / thuật toán RMS (`SMAC`…) |
| `Orders[]` / `Executions[]` (order replay) | lệnh (`OrderInfo`) / fills lịch sử — rỗng khi chưa giao dịch |
| `Dates[]` (OnOrderHistoryDates) | các ngày `CCYYMMDD` có lịch sử lệnh |
| `RpCode` ≠ 0 | **lỗi/không có data** cho query (vd `OnOptionList`=`7`) |
| `Price:"NaN"` | field có nhưng **chưa có giá trị** (band/indicator chưa công bố) |
| `Context` | object client gắn kèm request (gateway để `null`) |
| `DboUpdateType` (OnDbo) | **enum**: `Image` · `New` · `Change` · `Delete` · `Undefined` |
| `Type` (OnBar) | **BarType enum**: `Minute` · `Daily` · `Tick` · `Volume` · `Second` · `Range` · `Weekly` |

### L1 quote — `OnBidQuote` / `OnAskQuote` / `OnBestBidQuote` / `OnBestAskQuote` (symbol, push)
Cùng một shape (4 callback giống nhau, chỉ khác ngữ nghĩa mức giá):
```json
{ "e":"OnBidQuote", "CallbackType":"Update", "Exchange":"CME", "Symbol":"ESU6",
  "Price":7419, "Size":15, "NumOrders":13, "ImpliedSize":0, "LeanPrice":0,
  "UpdateType":"Solo|End|Undefined", "Ssboe":1782485821, "Usecs":152653, "Context":null }
```
### `OnBestBidAskQuote` (symbol, push — callback 2 đối số)
Bắn khi best bid **và** best ask đổi cùng lúc (không thường xuyên, ~20s). Là callback `(BidInfo, AskInfo)` → raw bus gói thành `{BidInfo, AskInfo}`:
```json
{ "e":"OnBestBidAskQuote",
  "BidInfo":{ "e":"BidInfo", "Exchange":"CME", "Symbol":"ESU6", "Price":7412.25, "Size":1,  "NumOrders":1,  "LeanPrice":7412.263, "UpdateType":"Undefined", "Ssboe":..., "Usecs":..., "Context":null },
  "AskInfo":{ "e":"AskInfo", "Exchange":"CME", "Symbol":"ESU6", "Price":7412.5,  "Size":18, "NumOrders":17, "LeanPrice":7412.263, "UpdateType":"Undefined", "Ssboe":..., "Usecs":..., "Context":null } }
```

> `OnQuote` / `OnQuoteReport` / `OnEndQuote`: **im trên CME** — Rithmic giao L1 qua các callback riêng ở trên, không dùng các callback tổng hợp này.

### `OnTradePrint` (symbol, push — mỗi lệnh khớp)
```json
{ "e":"OnTradePrint", "Exchange":"CME", "Symbol":"ESU6", "Price":7436.25, "Size":1,
  "AggressorSide":"S", "AggressorExchOrdId":"6417249122632", "ExchOrdId":"6417249122302", "Condition":"",
  "NetChange":13, "PercentChange":0.18, "Vwap":7400.75, "VwapLong":7400.83995214,
  "VolumeBought":0, "VolumeSold":334241,
  "SourceSsboe":1782487746, "SourceUsecs":213239, "SourceNsecs":213239655,
  "JopSsboe":1782487746, "JopNsecs":213505540, "Ssboe":1782487746, "Usecs":213594, "Context":null }
```
Field riêng: `AggressorSide` B/S (bên chủ động) · `AggressorExchOrdId`/`ExchOrdId` mã lệnh aggressor/khớp · `Condition` cờ điều kiện (rỗng=thường) · `NetChange`/`PercentChange` thay đổi ròng (điểm)/% so close trước · `Vwap`/`VwapLong` VWAP phiên (Long=chính xác cao) · `VolumeBought`/`VolumeSold` KL chủ động mua/bán luỹ kế.

### `OnTradeVolume` (symbol, push — tổng KL phiên)
```json
{ "e":"OnTradeVolume", "Exchange":"CME", "Symbol":"ESU6", "TotalVolumeFlag":true, "TotalVolume":670242,
  "SourceSsboe":..., "SourceUsecs":..., "SourceNsecs":..., "JopSsboe":..., "JopNsecs":..., "Ssboe":..., "Usecs":..., "Context":null }
```
`TotalVolume` = tổng KL phiên luỹ kế; `TotalVolumeFlag` = có giá trị hợp lệ.
> `OnTradeCondition` / `OnAggregator`: null trên future CME (chỉ bắn cho trade có cờ điều kiện / info aggregator — hiếm).

### Giá phiên / OHLC — `OnOpenPrice` · `OnHighPrice` · `OnLowPrice` · `OnClosePrice` · `OnHighBidPrice` · `OnLowAskPrice` · `OnMidPrice` · `OnCloseMidPrice` (symbol, push)

Đã xác minh: các flag tương ứng **đều nằm trong `SubscriptionFlags.All`** (mình subscribe bằng `All`) và các struct (`OpenPriceInfo`, `HighPriceInfo`, …) đều tồn tại → **gọi API đúng, không thiếu cờ**. **7/8 là field SNAPSHOT** — bắn `CallbackType:"Image"` một lượt khi subscribe mã mới tinh. Đã xác nhận trên NQU6 (cả 7 cùng `Ssboe:1782493440` = chung một snapshot, phục vụ từ cache):

```json
{ "e":"OnOpenPrice",    "CallbackType":"Image", "Symbol":"NQU6", "Price":29742    }
{ "e":"OnHighPrice",    "CallbackType":"Image", "Symbol":"NQU6", "Price":29892.75 }
{ "e":"OnLowPrice",     "CallbackType":"Image", "Symbol":"NQU6", "Price":29160.5  }
{ "e":"OnClosePrice",   "CallbackType":"Image", "Symbol":"NQU6", "Price":29759.25 }
{ "e":"OnHighBidPrice", "CallbackType":"Image", "Symbol":"NQU6", "Price":29892.25 }
{ "e":"OnLowAskPrice",  "CallbackType":"Image", "Symbol":"NQU6", "Price":29163.75 }
{ "e":"OnMidPrice",     "CallbackType":"Image", "Symbol":"NQU6", "LastPrice":"NaN","OpenPrice":"NaN",... }
```
| Callback | Flag | Ý nghĩa |
|---|---|---|
| `OnOpenPrice` | `Open` (0x40) | giá MỞ phiên |
| `OnHighPrice` / `OnLowPrice` | `HighLow` (0x100) | đỉnh / đáy phiên (giá khớp) |
| `OnHighBidPrice` / `OnLowAskPrice` | `HighBidLowAsk` (0x8000) | bid cao nhất / ask thấp nhất phiên |
| `OnClosePrice` | `Close` (0x8) | giá ĐÓNG phiên trước |
| `OnMidPrice` | `MidPrice` (0x4000) | **struct tóm tắt phiên** — `Price`/`LastPrice`/`OpenPrice`/`HighPrice`/`LowPrice`/`NetChange`/`PercentChange` (NaN khi chưa tính) |
| `OnCloseMidPrice` | `Close` 0x8 + `MidPrice` 0x4000 | **null thật trên future CME** — Rithmic không gửi close-midpoint cho future (dành cho instrument có bid/ask lúc đóng: cổ phiếu/option). Không nằm trong burst snapshot. |

#### Cache để hết RACE re-subscribe
Mấy cái này bắn **một lần** trong snapshot lúc subscribe đầu. Bấm "Thử" liên tiếp trên cùng mã thì `unsubscribe→subscribe` sát nhau khiến R|API+ **bỏ re-image** lệch nhịp (pattern xen kẽ ✓✗). **Fix:** gateway **cache giá trị mới nhất** của nhóm low-freq (OHLC/settlement/OI/market-mode/limit) ngay trong `OnAny` — **kể cả khi không ai xem** — rồi **đẩy cache cho client ngay khi subscribe** (`RawCachePush`). Nhờ đó lần Thử đầu mồi cả 8 vào cache, các lần sau lấy từ cache → ra đủ, không phụ thuộc re-image. (Đính chính: lúc đầu tưởng High/Low là "update-only không trong snapshot" — **sai**; tất cả đều là snapshot.)

#### Hành vi LIVE (giữ sub liên tục)
Lúc subscribe: cả 7 ra một lượt `CallbackType:"Image"`. Sau đó:
- `OnOpenPrice` / `OnClosePrice` — **tĩnh**, không bắn lại (giá mở / đóng phiên cố định).
- `OnHighPrice` / `OnLowPrice` / `OnHighBidPrice` / `OnLowAskPrice` / `OnMidPrice` — bắn `CallbackType:"Update"` **mỗi khi lập cực trị phiên mới** (đỉnh/đáy giá khớp, bid cao nhất, ask thấp nhất, mid). Đi ngang không phá đỉnh/đáy → im. Cache ghi đè theo mỗi `Update`.

### Settlement — `OnSettlementPrice` · `OnProjectedSettlementPrice` (symbol, push)
```json
{ "e":"OnSettlementPrice", "CallbackType":"Image", "Exchange":"CME", "Symbol":"NQU6",
  "Date":"20260625", "Price":29724.75, "PriceType":"final", "Ssboe":1782494266, "Usecs":0, "Context":null }
```
- `Date` = ngày settlement (CCYYMMDD). `PriceType` = `final` (chính thức) hoặc `preliminary` (sơ bộ).
- `OnProjectedSettlementPrice`: **null ngoài cửa sổ settlement**. CME chỉ phát settlement *dự kiến* gần giờ ĐÓNG phiên; ngoài khung đó chưa có gì để bắn (đã trong cache whitelist — quanh giờ settlement sẽ ra). Đối chiếu: `OnSettlementPrice` là bản `final` cố định nên luôn có trong snapshot.

### Limit & trạng thái chợ — `OnHighPriceLimit` · `OnLowPriceLimit` · `OnMarketMode` · `OnOpeningIndicator` · `OnClosingIndicator` (symbol, push)
```json
{ "e":"OnLowPriceLimit", "CallbackType":"Image", "Symbol":"NQU6", "Price":27663.75 }
{ "e":"OnHighPriceLimit","CallbackType":"Image", "Symbol":"NQU6", "Price":"NaN" }
{ "e":"OnMarketMode", "CallbackType":"Image", "Symbol":"NQU6",
  "MarketMode":"Open", "Event":"No Event", "Reason":"Group Schedule",
  "SourceSsboe":1782494595, "SourceNsecs":0, "JopSsboe":1782494595, "Ssboe":1782494595, "Usecs":0 }
{ "e":"OnOpeningIndicator", "CallbackType":"Image", "Symbol":"NQU6", "Price":"NaN", "VolumeFlag":false, "Volume":0 }
```
- `OnLowPriceLimit` / `OnHighPriceLimit` = band giá dưới / trên (limit-down/up). `Price:"NaN"` = band đó hiện **không công bố** (vd NQU6 chưa có band trên).
- `OnMarketMode` = **trạng thái phiên**. `MarketMode` (vd `Open`; còn `Closed`/`Pre-Open`/`Pause`/`Halt`/`Auction`…), `Reason` (vì sao — `Group Schedule` = theo lịch nhóm sản phẩm), `Event` (`No Event` = không có sự kiện).
- `OnOpeningIndicator` / `OnClosingIndicator` = **giá chỉ báo (IOP/ICP) trong đấu giá MỞ/ĐÓNG cửa**. Ngoài cửa sổ đấu giá → `Price:"NaN"`/`Volume:0` (OpeningIndicator) hoặc **null** (ClosingIndicator, như `OnProjectedSettlementPrice`). `VolumeFlag` = có khối lượng chỉ báo hợp lệ.

### Open interest — `OnOpenInterest` (symbol, push)
```json
{ "e":"OnOpenInterest", "CallbackType":"Image", "Symbol":"NQU6", "QuantityFlag":true, "Quantity":269932, "Ssboe":1782494927, "Usecs":0 }
```
`Quantity` = open interest (số hợp đồng đang mở); `QuantityFlag` = giá trị hợp lệ. Trong cache whitelist → Thử ra ngay.

### Book / DBO — `OnLimitOrderBook` · `OnDbo` · `OnDboBookRebuild` (symbol, push — feed qua `@depth` / `@mbo`)
**`OnLimitOrderBook`** = SỔ MBP đầy đủ (snapshot cả book, ~700+ mức **mỗi bên** — nguồn của DOM `@depth`). KHÔNG cache (quá lớn). Mỗi mức là 1 object như L1 quote:
```json
{ "e":"OnLimitOrderBook", "CallbackType":"Image", "Symbol":"NQU6", "RpCode":0,
  "Bids":[ {"Price":29646,"Size":1,"NumOrders":1,"ImpliedSize":0,"LeanPrice":0,"UpdateType":"Undefined","Ssboe":...,"Usecs":...}, … ],
  "Asks":[ {"Price":29646.75,"Size":1,"NumOrders":1,"UpdateType":"Undefined"}, … ] }
```
**`OnDbo`** = depth-by-order (MBO) — 1 sự kiện cho 1 LỆNH (nguồn của `@mbo`/`@mboraw`):
```json
{ "e":"OnDbo", "CallbackType":"Update", "Symbol":"NQU6", "ExchOrdId":"6878892898854",
  "DboUpdateType":"Delete", "Priority":"107428920692", "Side":"S", "Price":29651.5,
  "SizeFlag":true, "Size":1, "PreviousPrice":"NaN", "SourceSsboe":…, "SourceNsecs":…, "JopSsboe":…, "Ssboe":…, "Usecs":… }
```
- `DboUpdateType` = `Image`/`New`/`Change`/`Delete`. `ExchOrdId` = id lệnh trên sàn; `Priority` = thứ tự ưu tiên (FIFO) của lệnh trong hàng; `Side` = `B`/`S`; `PreviousPrice` = giá cũ khi lệnh đổi giá (NaN nếu không đổi).
- **`OnDboBookRebuild`**: **null bình thường** — chỉ bắn khi sổ DBO cần dựng lại toàn bộ (recovery/reset sau gap feed).

### Nến — `OnBar` · `OnBarReplay` (symbol, push — feed qua `@kline`)
```json
{ "e":"OnBar", "Symbol":"NQU6", "Type":"Minute", "SpecifiedMinutes":1,
  "OpenPrice":29634.25, "HighPrice":29646, "LowPrice":29615.75, "ClosePrice":29620.75,
  "Vwap":29486.5, "Volume":487, "NumTrades":464,
  "BuyerAsAggressorVolume":250, "SellerAsAggressorVolume":237,
  "ProfilePrices":[], "ProfileBuyerAsAggressorVolumes":[], "ProfileSellerAsAggressorVolumes":[],
  "StartSsboe":1782495240, "EndSsboe":1782495300 }
```
- `Type` = khung nến (BarType: `Minute`/`Tick`/`Volume`/`Second`/`Daily`/`Weekly`/`Range`). `Specified{Minutes,Ticks,Volume,…}` = thông số khung.
- `BuyerAsAggressorVolume` / `SellerAsAggressorVolume` = **delta footprint** (KL mua / bán chủ động). `Vwap` = VWAP phiên. `SettlementPrice:"NaN"` khi chưa settle.
- `Profile*[]` = footprint theo **từng giá** (rỗng ở `@kline` thường; có khi `replayBars(Tick)`). `OnBarReplay` = **null** lúc live — chỉ là cờ HOÀN TẤT khi replay nến lịch sử xong.

### Volume profile — `OnVolumeAtPrice` (symbol, RPC `volumeProfile`/getVolumeAtPrice)
```json
{ "e":"OnVolumeAtPrice", "Symbol":"NQU6", "RpCode":0,
  "Prices":[30263.75, 30263.5, …], "Volumes":[0, 0, …], "Context":"vp-<id>" }
```
Volume Profile **native**. `Prices[]` & `Volumes[]` **song song** (cùng index). Thật ~500+ mức. `Context` = id query echo lại.

### Reference / instrument — `OnRefData` · `OnAuxRefData` · `OnPriceIncrUpdate` (symbol, RPC `instrumentInfo`/`auxRefData`)
```json
{ "e":"OnRefData", "Symbol":"NQU6", "Description":"E-Mini Nasdaq-100", "Expiration":"20260918",
  "ProductCode":"NQ", "InstrumentType":"Future", "SinglePointValue":20, "Currency":"USD",
  "MaxPriceVariation":30, "ExpirationTime":"1789738200", "RpCode":0 }
{ "e":"OnAuxRefData", "Symbol":"NQU6", "FirstTradingDate":"20250321", "LastTradingDate":"20260918",
  "SettlementMethod":"cash", "UnitOfMeasure":"IPNT", "UnitOfMeasureQty":20, "RpCode":0 }
```
- `SinglePointValue` = giá trị 1 điểm ($20/điểm cho NQ). `Expiration`(CCYYMMDD) / `ExpirationTime`(epoch). Field option (`StrikePrice`/`PutCallIndicator`/`CapPrice`…) = `NaN`/`null` với future.
- `OnAuxRefData` = ngày notice/trading/delivery + `SettlementMethod` (`cash`/`physical`) + `UnitOfMeasure`.
- `OnPriceIncrUpdate` = **bảng bước giá / tick-size**:
```json
{ "e":"OnPriceIncrUpdate", "Symbol":"NQU6", "RpCode":0,
  "Rows":[ {"FirstPrice":0,"LastPrice":0,"PriceIncr":0.25,"FirstOperator":"None","LastOperator":"None","Precision":2} ] }
```
  `PriceIncr` = tick (0.25 NQ); `Precision` = số chữ số lẻ; `FirstPrice`/`LastPrice` = khoảng giá áp dụng (`0`/`0` = mọi giá); sản phẩm **tick biến thiên** có nhiều `Row`. Gateway tự gọi `getPriceIncrInfo` khi subscribe `@onpriceincrupdate` (bỏ qua dedup của `instrumentInfo`) + cache lại.

### Instruments / options / strategy — `OnExchangeList` · `OnInstrumentSearch` · `OnInstrumentByUnderlying` · `OnOptionList` (global, query-response)
```json
{ "e":"OnExchangeList", "Exchanges":["COMEX","NYMEX","CBOT","CME"], "RpCode":0 }
{ "e":"OnInstrumentSearch", "Exchange":"CME",
  "Terms":[ {"Field":"Symbol","Operator":"Contains","Term":"NQ","CaseSensitive":false} ],
  "Instruments":[ {"Symbol":"NQU6","InstrumentType":"Future",...}, {"Symbol":"NQM9 C25000","InstrumentType":"Future Option","PutCallIndicator":"Call","StrikePrice":25000,"Underlying":"NQM9",...} ],
  "RpCode":0, "Context":"search-<id>" }
{ "e":"OnInstrumentByUnderlying", "Underlyings":["NQU6"], "Exchanges":["CME"],
  "Expirations":["20260624","20260625",…], "Instruments":[], "RpCode":0 }
{ "e":"OnOptionList", "Instruments":[ {"Symbol":"NQU6 C29000","StrikePrice":29000,"PutCallIndicator":"Call","Underlying":"NQU6",…} ], "RpCode":0 }
```
- `OnInstrumentSearch.Instruments[]` = **hàng nghìn** mã khớp (mỗi phần tử = shape `OnRefData`). `Terms[]` = tiêu chí tìm. `InstrumentType` = `Future` / `Future Strategy` / `Future Option`. Option: `PutCallIndicator`(`Call`/`Put`) · `StrikePrice` · `Underlying`. Mã liên tục (`NQ`) có `TradingSymbol`(=`NQU6`).
- `OnInstrumentByUnderlying` = liệt kê `Expirations[]` option theo underlying. `OnOptionList` = chuỗi option theo product + **expiration** (`YYYYMM`).
- **`RpCode`**: `0` = OK; **`≠0` = lỗi/không có data**. ⚠️ `optionChain` mặc định `expiration`=THÁNG NÀY → trúng option đã hết hạn → `7`. Truyền đúng tháng của future (NQU6 → `202609`) thì ra `0` + chuỗi.

### Strategy / spread / trade-route — `OnStrategyList` · `OnStrategy` · `OnBinaryContractList` · `OnTradeRouteList` · ETB (global, query-response)
```json
{ "e":"OnStrategyList", "Products":["NQ"], "StrategyTypes":["Futures Calendar","Iron Condor","Options Straddle","Risk Reversal",…], "RpCode":0 }
{ "e":"OnTradeRouteList", "RpCode":0,
  "TradeRoutes":[ {"Exchange":"CME","FcmId":"FCM1","IbId":"IB1","TradeRoute":"simulator","Status":"UP","Default":"Y"} ] }
```
- `OnStrategyList` = các **loại** spread hỗ trợ (`StrategyType`). `OnStrategy` (strategyInfo) cần product + `StrategyType` + `Legs` cụ thể → thiếu thì `RpCode 7`.
- `OnTradeRouteList` = **route đặt lệnh** (1/sàn). `FcmId`+`IbId`+`TradeRoute` định tuyến order; `Status` `UP`/`DOWN`; `Default` `Y`. `simulator` = tài khoản demo. **Cần cho order entry.**
- `OnBinaryContractList`: `RpCode 7` cho NQ (binary/event nằm ở product riêng vd `ECNQ`). `OnEasyToBorrowList`: `RpCode 0` nhưng rỗng (ETB chỉ cho cổ phiếu). **null** (event/write-only): `OnUserDefinedSpreadCreate` (chỉ khi tạo UDS) · `OnEasyToBorrow` (cổ phiếu) · `OnTradeRoute` (push cập nhật 1 route).

### Account / PnL / RMS — `OnPnlReplay` · `OnProductRmsList` (global, query-response)
> **STATE-WRAPPER** (đặc thù Rithmic): nhiều field tiền tệ là object `{Ignore, Clear, Use, State, Value}`. `State` = `Use` (dùng `Value`) · `Clear` (đã xoá → `Value:null`) · `Ignore` (không đổi). Đọc `Value` **chỉ khi** `State:"Use"`.
```json
{ "e":"OnPnlReplay", "RpCode":0,
  "Account":{ "FcmId":"FCM1","IbId":"IB1","AccountId":"sample@abc.com","AccountName":"Demo Account",
              "RmsInfo":{"Algorithm":"SMAC","Currency":"USD","Status":"active","LossLimit":"NaN","MaxOrderQty":0} },
  "PnlInfoList":[ { "Exchange":null,"Symbol":null,
                    "AccountBalance":{"State":"Use","Value":100070}, "CashOnHand":{"State":"Use","Value":100070},
                    "OpenPnl":{"State":"Use","Value":0}, "ClosedPnl":{"State":"Use","Value":0}, "Position":{"State":"Use","Value":0} } ] }
{ "e":"OnProductRmsList", "RpCode":0,
  "Products":[ {"Account":{…,"AccountId":"sample@abc.com"},"ProductCode":"GC","BuyLimit":0,"SellLimit":0,"MaxOrderQty":0,"LossLimit":"NaN"} ] }
```
- `OnPnlReplay.PnlInfoList[]` = 1 dòng **tổng account** (`Exchange`/`Symbol` `null`) + 1 dòng **/mã có vị thế**. Field: `AccountBalance`/`CashOnHand`/`MarginBalance` · `OpenPnl`/`ClosedPnl` · `Position`/`BuyQty`/`SellQty`/`LongExposure`/`ShortExposure` (đều state-wrapper). `Account.RmsInfo` = cấu hình rủi ro (`Algorithm`, `LossLimit`, `MaxOrderQty`).
- `OnProductRmsList` = hạn mức RMS **theo product**: `BuyLimit`/`SellLimit` (giới hạn vị thế), `MaxOrderQty`, `LossLimit`, margin rate. `*Flag:false` → hạn mức đó không áp.
- **null** (event/login/write): `OnAccountList` (login-time → RPC `account`) · `OnAccountUpdate`/`OnPnlUpdate` (cần subscribe + biến động) · `OnPositionExit` (exitPosition) · `OnAutoLiquidate` (sự kiện auto-liq) · `OnSodUpdate` (đầu ngày).

### Lệnh — replay + report — `OnOrderReplay` · `OnOpenOrderReplay` · `OnExecutionReplay` · `OnOrderHistoryDates` (global, query-response)
> Account id trong mọi callback đã **ẩn** (`sample@abc.com`). `Account` block = giống `OnPnlReplay` (kèm `RmsInfo`/`AutoLiquidateInfo`).
```json
{ "e":"OnOrderReplay", "RpCode":0, "Date":"", "Account":{…,"AccountId":"sample@abc.com"}, "Orders":[] }
{ "e":"OnOpenOrderReplay", "RpCode":0, "Account":{…}, "Orders":[] }
{ "e":"OnExecutionReplay", "RpCode":7, "Account":{…}, "Executions":[] }
{ "e":"OnOrderHistoryDates", "RpCode":0, "Dates":["20111001",…,"20130304"] }
```
- `OnOrderReplay` (replayAllOrders, theo khoảng) / `OnOpenOrderReplay` (lệnh đang mở): `Orders[]` = `OrderInfo[]` (giá/size/side/trạng thái) — **rỗng** khi tài khoản chưa đặt lệnh. `OnExecutionReplay`: `Executions[]` = fills lịch sử (`RpCode 7` khi rỗng).
- `OnOrderHistoryDates` = các **ngày** (`CCYYMMDD`) có lịch sử lệnh → dùng làm tham số `replayHistoricalOrders`. (Demo có data 2011→2013.)
- **null** (cần lệnh thật / orderNum): `OnSingleOrderReplay` (1 lệnh — kích bằng `replaySingleOrder{orderNum}` hoặc `replaySingleHistoricalOrder{orderNum,date}`), `OnLineUpdate` (cập nhật 1 dòng lệnh khi có lệnh hoạt động).
- **Bracket** (đều null khi chưa có bracket order): `OnBracketReplay` (query-response `replayBrackets` — bracket OCO/SL+TP lịch sử), `OnBracketUpdate` (1 bracket LIVE đổi), `OnBracketTierModify` (sửa tier SL/TP).
- **Report** (đều null — event thuần, KHÔNG snapshot; chỉ bắn theo vòng đời lệnh THẬT, cần `subscribeOrders` + đặt lệnh route `simulator`): `OnStatusReport` (trạng thái), `OnFillReport` (khớp), `OnModifyReport`/`OnCancelReport` (sửa/huỷ OK), `OnNotModifiedReport`/`OnNotCancelledReport` (sửa/huỷ bị reject), `OnRejectReport` (lệnh bị từ chối), `OnFailureReport` (gửi lỗi), `OnBustReport` (huỷ khớp), `OnTradeCorrectReport` (sửa trade đã khớp), `OnTriggerReport`/`OnTriggerPulledReport` (stop kích hoạt/bị rút), `OnOtherReport` (catch-all). Account demo chưa đặt lệnh → tất cả im; đặt 1 lệnh sim để thấy `OnStatusReport`→`OnFillReport`.

### User / IB / agreement — `OnUserProfile` · `OnAgreementList` · `OnUserList` · `OnIbList` · ETB (global, query-response)
> ⚠️ `OnUserProfile` chứa **PII** (tên/địa chỉ/điện thoại/email) → trong tài liệu đã **ẩn** (`sample@abc.com` / `***`). Phần giữ lại có giá trị = **đếm số phiên** mỗi plant.
```json
{ "e":"OnUserProfile", "User":"sample@abc.com", "UserType":"3", "FirstName":"Sample","LastName":"User","Email":"sample@abc.com",
  "CurrentMarketDataSessionCount":1, "CurrentOrdersSessionCount":2, "CurrentPnlSessionCount":1, "CurrentHistorySessionCount":1,
  "PeakOrdersSessionCount":2, "PeakPnlSessionCount":2, "PeakRepositorySessionCount":1,
  "MaxMarketDataSessionCount":1, "MaxOrdersSessionCount":4, "MaxHistorySessionCount":0, "MaxPnlSessionCount":0, "MaxRepositorySessionCount":0,
  "RpCode":0, "ConnId":"TradingSystem" }
{ "e":"OnAgreementList", "Accepted":false, "Agreements":[], "RpCode":0 }
```
- `OnUserProfile` (`userProfile`) — `Current/Peak/Max {MarketData,Orders,Pnl,History,Repository}SessionCount`. **`Max` = giới hạn phiên đồng thời / 1 USER** (`MaxMarketData=1` → mỗi user chỉ 1 phiên market-data; `MaxOrders=4`; `Max=0` = không đặt/không giới hạn). `Current` = đang mở (Orders=2: app này + R Trader Pro chạy song song). 👉 **Đa-login:** mỗi *user* có hạn phiên riêng → 2 user khác nhau OK; cùng 1 user dễ đụng trần (đặc biệt MarketData=1).
- `OnAgreementList` (`listAgreements(accepted)`) — `accepted=false` + rỗng = không còn thoả thuận **chưa** ký (đã ký hết); gọi `accepted=true` để liệt kê đã ký. (Login fail vì chưa ký → `OnUnsignedAgreement` ở LoginForm.)
- **null** (tài khoản thường, không phải IB-admin): `OnUser` (push qua `subscribeUser(ibId)`), `OnUserList` (`listUsers` user dưới 1 IB), `OnIbList` (`listIbs` IB con), `OnAssignedUserList` (`assignedUsers` user gán vào account), `OnPasswordChange` (chỉ bắn khi gọi `changePassword`).

### `OnEnvironmentList` (global, RPC `listEnvironments`)
```json
{ "e":"OnEnvironmentList", "Environments":["system"], "Context":null }
```

## 6. Changelog

- **2026-06-28** — Pull RPC names switched to the **REngine method** (`listExchanges`, `getOptionList`, `searchInstrument`…); old Binance-style names kept as dispatch **aliases** (chart/api.html unaffected). Exposed `replaySingleHistoricalOrder` (orderNum+date → `@OnSingleOrderReplay`). §3 rewritten as full Query-inline / Trigger tables. New **examples.html** — client code in 9 languages (Python/C#/JS/TS/C++/Java/Go/Rust/Kotlin) for stream + RPC-inline + RPC-trigger. (api.html keeps Binance-style names = the composite API layer; raw = REngine names.)
- **2026-06-26** — Raw bus: every R|API+ push callback is now a 1-to-1 stream (`@<CallbackName>`, global via `*.*@…`) via a reflective `OnAny` passthrough; added pull RPCs `listIbs` / `listUsers` / `userProfile` / `listAgreements`.
