# Hidden Coupling

> Source: [Top Types of Hidden Coupling and How to Deal With Them — ByteByteGo](https://blog.bytebytego.com/p/top-types-of-hidden-coupling-and)（2026-11-13）

## TL;DR

- 架構圖只畫出**靜態結構**（誰呼叫誰）；hidden coupling 多半是**執行期與運作特性**，要在「改東西」或「出事」時才會現形。
- 耦合無法消除，目標是讓它 **清楚、有限、穩定**：一般的改動不應該牽動一大片。
- 拆成微服務不會自動帶來獨立性：商業規則的 ownership、共用的 table、不相容的介面、共用的基礎設施都還在，反而多了網路延遲與新的失敗模式。

## 五個檢視面向

| # | 問題 | 面向 | 對應類型 |
|---|---|---|---|
| 1 | 兩個元件必須**同時在線**嗎？ | 執行期 / 時間 | Temporal |
| 2 | 兩者對同一資料或事件的**解讀**必須一致嗎？ | 語意 | Semantic |
| 3 | **改一個**就得改好幾個嗎？ | 變更 | Shared schema / library、Observability |
| 4 | 必須**協調多個版本**一起上線嗎？ | 部署 | Deployment |
| 5 | 能**單獨復原**，不必連帶復原其他元件嗎？ | 復原 | Shared incidents |

**這五個面向彼此獨立。**

- 可各自部署、但每個 request 都互相呼叫：#4 無耦合，#1 耦合重。
- 反例：透過 Kafka 溝通的 producer / consumer，或共讀同一個 DB 欄位。執行期互不呼叫（#1 無耦合），但事件格式或欄位單位（例如秒改成分）一改，就得一起改、一起上線（#2、#3、#4 都有耦合）。綁住它們的是**中間那份資料格式**。

## 1. Temporal Coupling

一個元件能否正常運作，取決於另一個元件**何時**執行或回應。

**可用性型**：註冊流程同步等待 email service 寄出歡迎信。email 一慢，註冊就變慢；email 一掛，就沒人能註冊。

- 前端類比：render 被 feature flag / remote config 卡住。解法是預設值、快取上次的 config、timeout，讓 render 不必等它。

**順序型**：2:00 跑 export、2:05 跑 import，前提是 export 永遠在 5 分鐘內完成。資料一長大，import 就默默讀到不完整的資料。

- 用時間排程頂替了明確的**完成訊號**。
- 前端類比：`setTimeout` 賭 service worker 已就緒 → 應該用 `navigator.serviceWorker.ready` / `controllerchange`。

**解法**

- **Durable message**：producer 記下工作即可回應，consumer 晚點處理。
- **Transactional outbox**：商業資料與 pending event 寫在**同一個 transaction**，再由 relay 發送。
  - relay 的流程：讀取 pending → publish → 標記為 sent。若 publish 成功、標記前 crash，重啟後會再送一次 → **at-least-once**。
  - consumer 必須 **idempotent**（例如用 `event_id` 去重）。
- **排程**：改用明確的 readiness signal 觸發下游。

**限制**：queue 需要處理儲存、保留時間、處理能力；也要看業務時限。

- 歡迎信：使用者不會等 → 可以完全解耦。
- OTP 驗證信：使用者正在等，**時效本身就是需求** → 保留耦合但讓它可控（獨立的高優先 queue、SLA、timeout、備援供應商）。

## 2. Semantic Coupling

元件對同一份資料的**意義**理解不同。只對齊欄位名稱與型別不夠。

- `available_quantity`：倉庫認為是實際庫存，前台認為是可售數量。有預留時就會超賣。
- 交易所的 `balance`：total = available + locked。只回一個 `balance`，下單頁和資產頁可能各自解讀，一掛單就出現不一致。
- **同一條商業規則實作在多處**：premium 判斷同時寫在 subscription 與 content service，寬限期一改就得改兩處，改得不一致就會出錯。code 分開了，**規則的 ownership 沒有分開**。

**解法**

- **Bounded context**（DDD）：context 內詞彙意義一致，跨 context 溝通要明確轉換。
- **明確的契約命名**：`stock_on_hand` / `stock_available_to_sell`。
- **每條商業規則要有 owner**，其他人向 owner 取得：
  1. 直接拿結果（`canTrade: false, reason: 'MIN_NOTIONAL'`）
  2. 拿規則參數做 **local projection**（例如 `exchangeInfo` 的 `minQty`、`tickSize`）
- **Anti-Corruption Layer**：在邊界用 adapter 把外部模型轉成內部模型。
  - 成本：多一層 mapping、兩份型別、追 code 時多跳一層。
  - **用在你控制不了的邊界**（第三方 / legacy API）；內部同團隊的 API 比較適合 contract-first（OpenAPI 產生型別）。

## 3. Deployment Coupling

改一個元件時，必須跟其他元件協調上線。

- 例子：`delivery_address` 改名為 `shipping_address`，新舊 consumer 並存期間一定有人壞掉。
- Joint release 會變成例行負擔：小改動也要一起測、一起上線、一起 rollback。

**Expand → Migrate → Contract**

```
1. Expand    同時回傳新舊兩個欄位
2. Migrate   各 consumer 依自己的節奏改讀新欄位
3. Contract  確認沒人使用舊欄位後才移除
```

把一個不相容的改動，拆成三個各自相容、可單獨回退的小步驟。

- 前提：consumer 是 **Tolerant Reader**，會忽略不認識的欄位。
  - 會破壞這個前提的例子：Jackson 預設 `FAIL_ON_UNKNOWN_PROPERTIES = true`。
  - 前端解析 JSON 本來就會忽略多餘欄位；前端真正的痛點是**舊版 client 還在使用者手上**（沒重新整理的分頁、沒更新的 app），讓 Contract 遲遲無法執行。

## 4. Shared Schemas & Libraries

為了重用而共享，卻把決策擴散到 owner 以外的地方。

### Shared schema

support service 直接查 billing 的 table → 依賴 billing 的**內部 DB 設計**，billing 改欄位、拆 table 就壞掉。

| | 變更耦合 | 執行期耦合 | 資料新鮮度 |
|---|---|---|---|
| 直接讀 table | 高 | 中（共用 DB） | 即時 |
| Owner 提供 API / event | 低 | **高** | 即時 |
| Consumer-owned read model | 低 | 低 | **可能過時** |

同一對 service，不同的查詢可以選不同方式：

- 「這筆付款剛剛成功了嗎？」→ 需要即時 → API
- 「上個月退款過的客戶名單」→ 可以接受延遲，且避免把大查詢壓力丟給 owner → read model

### Shared library

- 一人改 library，所有人都受影響；若每次都得一起升級，library 就成了另一個需要協調上線的理由。
- 緩解：版本管理、讓 consumer 各自升級、保持範圍窄小。
- **但「各自升級」對商業規則會失效**：寬限期規則 v2 只有部分 service 升級 → 同一個使用者在不同頁面得到不同的 premium 判斷。
- 判斷標準不是名字叫 helper 還是 rule，而是**輸出有沒有參與商業決策**。只用於顯示的 `formatDate` 版本不一致無妨；若同一個日期函式被拿去算到期時間，它就已經是規則的一部分。
- → 商業決策集中在 owner service 計算，library 只放穩定、不帶商業意義的工具。

## 5. Observability Coupling

監控變成依賴鏈的一部分。

**Telemetry 擋在 request 路徑上**

- 同步送 log：collector 一慢，request thread 就卡住。
- 無上限的 buffer：collector 長時間掛掉時塞爆記憶體或磁碟。
- 原則：限制資源用量、隔離失敗（非同步、buffer 設上限、滿了就 drop）。

**Telemetry 本身就是契約**

- alert 比對 log 字串 `"payment failed"`，改成 `"payment unsuccessful"` 後 alert 默默失效。
- metric 改名，autoscaling 還在用舊名稱 → 影響 production 容量。
- 屬於**變更**加上**語意**耦合：寫 log 的人以為是給人看的文字，alert 卻當作機器讀取的訊號。
- 解法：盤點哪些欄位被 alert、dashboard、自動化工具使用；用結構化欄位（`event: "payment_failed"`）取代自由文字；被機器讀取的欄位比照 API，改動時走 expand → migrate → contract。

## 6. Shared Incidents

沒有直接溝通，卻在同一個 **failure domain**（會被同一個底層故障一起影響的元件集合）。

- 例子：分析 workload 與認證 service 用同一台 DB 的不同 schema，大型 query 吃光 CPU / IO，登入跟著變慢。schema ownership 分得很清楚，**資源卻沒隔離**。
- 常見的共用點：cluster、網路路徑、DNS、credential provider、config 系統、cloud region。

| | Common failure | Cascading failure |
|---|---|---|
| 意義 | 同一個原因讓多個元件**一起倒** | 故障沿呼叫鏈**擴散並放大**（骨牌接連倒） |
| 例子 | Redis 掛掉，所有使用者同時報錯；分析 query 拖慢登入 | Redis 變慢 → X 的 thread pool 塞滿 → 呼叫 X 的 Y timeout；登入變慢 → 大量 retry → DB 更慢 |

- 緩解：依重要程度決定隔離的程度，例如分開 connection pool、限制並發、預留容量、設定 workload quota。

## 找到依賴之後的處理選項

- **移除**不必要的依賴
- 把隱藏的契約**變成明確的**
- **降低**它變動的頻率
- 故障時**控制**影響範圍
- **接受**：共用平台可能比維護多份獨立實例便宜
- **重劃邊界**：兩個 service 一直交換內部細節、需要即時一致、總是為同樣的原因一起改 → 合併成一個結構良好的 module

**練習：共用 logging 系統**

- 不移除、不合併，因為它本來就該共用。
- 明確化：被 alert 使用的欄位定義成 schema。
- 降低變動：機器讀取的欄位視為契約。
- 控制影響：非同步、buffer 設上限、滿了就 drop。
- 接受：共用才能追蹤跨 service 的 user journey。

## 延伸閱讀

- Michael Nygard,《Release It!》：stability patterns（bulkhead、circuit breaker、timeout）
- DDD：bounded context、context mapping、anti-corruption layer
- Transactional outbox / idempotent consumer
