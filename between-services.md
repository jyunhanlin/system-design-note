# Between Services

> Source: [The Invisible Systems: What Happens Between Services — ByteByteGo](https://blog.bytebytego.com/p/the-invisible-systems-what-happens)（2026-11-20）

## TL;DR

- 架構圖上的**箭頭本身就是一個系統**：連線、地址、時間預算、retry 策略都有各自的成本與失敗模式。兩端服務都健康，request 仍可能失敗。
- **沒收到 response ≠ 操作失敗**：timeout 只代表 caller 不等了，server 端的狀態是未知的。
- 各種問題會互相放大：慢依賴 → pool 耗盡 → timeout → 各層 retry 相乘 → 更慢。

## 1. Service call 是一段旅程

同一個 process 內的 function call 只是轉移控制權；跨服務呼叫即使 client library 包裝得像 function，背後是：

```
serialize → 解析地址 / 取得連線 →（新連線：TCP + TLS handshake）
→ LB / proxy → 對方 decode + 業務邏輯 → response 走回程
```

- **一次性成本 vs 每次成本**：建連線只在第一次付，serialize 與傳輸每次都付 → 啟動後第一個 request 常常特別慢。
- TCP 保證**有序與重傳**，不保證**時效**；重傳會拉高延遲，TCP 也不會持續確認對端是否還活著。
- **Timeout = 結果未知**。可能是 (A) 根本沒送到、(B) 還在處理、(C) 已完成但 response 在回程遺失，caller 無法分辨。
  - 例：checkout 呼叫 payment 扣款 timeout 後直接 retry → 可能**重複扣款**。應標記為 pending、查詢實際狀態，或使用 idempotency key。
- **Partial failure**：同一服務的某個 instance 連線壞了，其他 instance 正常。

## 2. 搬資料的成本

- Serialize / deserialize 會吃**兩端**的 CPU 與記憶體：暫存 buffer、資料複製、object allocation → GC。成本遠超過傳輸時間本身。
- 例：BFF `/products` 回傳 500 筆完整物件（約 2MB JSON），前端列表只用 `name`、`price`。DB query 10ms，但 p99 很高、高峰時 GC pause 變多。

| 手段 | 改善了什麼 | 沒解決什麼 |
|---|---|---|
| 開 gzip | 傳輸的 bytes（p99 中網路傳輸那段） | 用不到的欄位照樣建構、serialize；GC 壓力還在，還**多了**壓縮的 CPU 與 buffer |
| 只回需要的欄位 / 分頁 / 限制 response 大小 | 從源頭減量 | — |

- **格式**：JSON 每筆都帶文字欄位名稱；Protobuf 用 schema + 編號欄位的 binary 編碼，通常更小更快，但效益取決於資料、語言與實作，要量測。
- **壓縮**：大且重複的資料很划算；小訊息或已壓縮的內容（圖片）幾乎沒有好處。
- 量測**整條路**：建構訊息、編碼、傳輸、記憶體，而不只是 query。

## 3. 兩邊都健康，卻對不上

通訊成功需要在多個層級達成共識：

| 層級 | 要對齊什麼 | 出事的樣子 |
|---|---|---|
| Protocol | 通訊規則 | gRPC 走 HTTP/2；為 HTTP/1.1 JSON 設定的 proxy 不一定能正確轉發 |
| 「成功」的定義 | 哪一層說了算 | gRPC 可能 **HTTP 200 但 RPC 失敗**，真正結果在 trailer 的 `grpc-status` → 只看 HTTP status 的 dashboard 會一片綠 |
| Schema | 結構與型別 | number 改成 string，舊 consumer 拒收；新增欄位要看接收方能否容忍未知欄位 |
| Semantics | 意義（單位、預設值、順序、缺值） | 一邊的 30 是秒、另一邊當毫秒。**兩邊都 parse 成功、沒有任何 error log**，行為卻錯了 |

- **部署期最危險**：rolling deploy 時新舊版本並存；rollback 會延長並存期，甚至讓舊 consumer 回來。
- 對策
  - 只做**相容的新增**。例：`retryAfter` 要改成毫秒 → 新增 `retryAfterMs`，舊欄位維持原意，等 consumer 都遷移後再 deprecate（同 [Hidden Coupling](./hidden-coupling.md) 的 expand → migrate → contract）。
  - 明確定義欄位的意義（單位要寫進名稱或契約）。
  - 測試**會同時存在的版本組合**。
  - Integration test 要走**真實的 proxy 與傳輸路徑**；全部 mock 掉就測不到 proxy 剝掉 metadata 或協商錯 protocol。

## 4. Timeout 與 Deadline

- **Timeout** 是一段時長（最多等 500ms）；**Deadline** 是一個時間點（過了結果就沒用）。
- **每次下游呼叫都在花 caller 剩下的時間**：下游的時間必須塞進上游剩餘的預算，並保留處理結果、回傳的時間。整條鏈共用同一個絕對 deadline 最直觀。

```
gateway deadline 1000ms
order-service 自己的工作 600ms → 呼叫 inventory，寫死 timeout 800ms
inventory 花 700ms → 總共 1300ms
→ 使用者在 1000ms 就收到失敗，order-service 最後 300ms 白等
→ 應該只給 inventory「剩下 400ms − 自己回傳所需」
```

| 計時器 | 限制什麼 |
|---|---|
| Connection timeout | 建立連線 |
| Pool acquisition timeout | 從 pool 取得可用連線 |
| Idle timeout | 無活動的時間 |
| Total request deadline | 整個操作 |

> 叫做「request timeout」的設定不一定涵蓋所有階段。

**Cancellation**

- 不等 ≠ 停止工作。取消要**往下游傳遞**，程式碼要主動檢查取消狀態並釋放資源；也**不會 rollback** 已 commit 的變更。
- 沒處理的後果：使用者看到失敗連按 3 次重新整理 → inventory 上同時跑 4 個 request，只有最新 1 個有人要 → 更慢 → 更多 timeout → 更多重新整理。
- 前端類比：`AbortController` 只停止瀏覽器端的等待；server 會不會停工，取決於它有沒有偵測到 client 斷線並取消下游。

## 5. Retry 會相乘

- Retry 適合**暫時性故障**（connection reset、instance 短暫不可用），但每次 retry 都在系統最吃緊時增加負載。
- **各層獨立 retry 會相乘**：前端 fetch wrapper 3 次 × BFF HTTP client 3 次 × service mesh sidecar 3 次 → 最底層最多被打 **27 次**。很多 retry 不在自己的 code 裡（library、proxy、mesh）。

| 邊界 | 範圍 | 作用 |
|---|---|---|
| Attempt limit | **單一 request** | 最多試幾次 |
| Time budget | 單一 request | 所有嘗試 + backoff 都要塞進原本的 deadline |
| Traffic budget | **一段時間內所有 request** | 例：每 1000 個原始 request 最多 100 次額外嘗試 |

- 服務全掛的一分鐘進來 10,000 個 request：只有 attempt limit = 3 → **30,000** 次；加上 10% traffic budget → 最多 **11,000** 次。
- **Backoff + jitter 只改變 retry 的時間分布**（拉開、打散），**不減少總次數**。
- **Idempotency** 解決的是「重複做安不安全」，與負載無關：idempotency key + server 端紀錄，讓重複 request 對應到同一個邏輯操作。
- 只對適合的錯誤 retry；retry 沒有提升成功率、只在消耗資源時就停。

## 6. 慢依賴耗光 Connection Pool

- Pool 讓連線可重用，但也是**容量上限**：連線用光時，新 request 連送出去的機會都沒有。
- **Little's Law：同時佔用的連線數 ≈ rps × 平均耗時**

```
pool = 50，流量 200 rps
正常 50ms → 200 × 0.05 = 10 條
變慢 2s   → 200 × 2    = 400 條 → 排隊或被拒絕
```

- 下游變慢 → 上游故障：排隊的 request 佔記憶體 → 延遲上升 → timeout → retry（接回第 4、5 段）。
- HTTP/2 一條連線可跑多個 stream，容量模型不同，但 stream 數量仍有上限。
- **把 pool 調大**：下游有餘力才有用。若下游是因為**它自己的 DB 出問題**而變慢，更大的 pool 只會把更多並發工作推進瓶頸，BFF 自己也要撐住大量在途 request。
- 對策
  - 等待要有上限（bounded waiting）。
  - Deadline 要**包含在本地 queue 等待的時間**。
  - Concurrency limit 反映下游真正能承受的量；必要時降級（例如不顯示推薦區塊）。
  - Response body / stream 一定要讀完或關閉，資源才會釋放。
  - Async 只減少被卡住的 thread，等待中的操作**仍佔用連線、buffer、記憶體**。

## 7. DNS 變更不會立刻生效

兩件容易混淆的事：

1. **解析結果被 cache**：TTL 規定 DNS 答案能 cache 多久，但 resolver、OS、runtime、client library 各有規則，光看 TTL 推不出 app 多久會重新解析。
2. **既有連線不會重查 DNS**：連線建立時就決定了目的地，DNS 改了也不會把它移到新地址。

- 搬遷時：新啟動的 client 連到新機器，老 client 繼續打舊機器，**兩邊都照設定正常運作**。
- 例：`inventory.internal` 改指向新 cluster、TTL 60 秒，10 分鐘後舊 cluster 仍收 30% 流量 → 查詢被 cache、既有連線沒重查。
- 不要走極端（關掉 keep-alive、每個 request 開新連線）：handshake 成本 + 耗盡本地網路資源。
- 方向：**有控制地重用、有控制地替換**
  - 設定連線最大壽命（max connection age），換新時自然重新解析。
  - 舊端主動 **draining**：停止接新工作、讓既有工作在有限時間內完成（HTTP/2 可送 `GOAWAY`）。

## 8. 連線會悄悄死掉

- 閒置連線在本地看似開著，實際上路徑已斷：對端機器消失沒有正常關閉、NAT / 防火牆把連線狀態丟掉。要等到送出流量或做 liveness check 才會發現。
- gRPC 的 **channel** 是 client 端抽象，物件還在不代表底下的連線可用。
- **HTTP/2 keepalive**：定期送 `PING`，時限內沒收到 ACK 就判定連線已壞。

**案例：防火牆閒置 5 分鐘就悄悄丟掉連線狀態，早上第一批 request 都卡到 timeout**

1. Pool 借出一條「本地看起來正常」的死連線。
2. 封包到了防火牆被直接丟掉，沒有 RST，client 收不到任何回應。
3. TCP 以為是掉封包，不斷重傳且間隔越拉越長 → 只能等 caller 自己 timeout。之後換新連線就恢復。

→ 解法：keepalive 間隔**短於防火牆的閒置門檻**（例如 1–2 分鐘），讓連線不會被判定為閒置，萬一死了也能在 request 進來前先發現。

**Keepalive 的限制**

- **連線活著 ≠ 應用健康**：對端可以正常回 ping，同時 handler 卡住或 DB 掛了。
- **ping 只測到對端**：中間有 proxy（例如 Envoy sidecar）終止連線時，只測到 client → proxy 這一段。
- → 仍需要 application-level 的 health check。

## 串起來：放大迴圈

```
依賴變慢 → pool 耗盡、排隊 → timeout
   ↑                              ↓
沒取消的殭屍工作 ← 各層 retry 相乘 + 使用者重新整理
```

## 延伸閱讀

- Circuit breaker、load shedding、adaptive concurrency limit：切斷上面這個迴圈的機制
- gRPC deadline propagation、keepalive 設定
- Little's Law
- [Hidden Coupling](./hidden-coupling.md)：Shared incidents / cascading failure
