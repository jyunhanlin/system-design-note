# Observability Basics

> Source: [Observability for Beginners: Logs, Metrics, Traces, and Everything Around Them — ByteByteGo](https://blog.bytebytego.com/p/observability-for-beginners-logs)（2026-11-14）

## TL;DR

- Logs、metrics、traces 不是三種不同的資料，而是**同一串 event** 在不同聚合層級的三種投影。
- **Cardinality** 決定什麼能放進 metrics：每一種 label 組合就是一條 time series。`user_id` 這類無上限的維度要放 logs / traces。
- Distributed tracing 靠兩件事：跨服務傳遞的 **trace ID**（context propagation），以及決定留哪些 trace 的 **sampling**。
- **Monitoring** 檢查事先想到的問題；**observability** 讓你問事先沒想到的問題。兩者都要，誰都不能取代誰。
- Alert 要打在使用者感受得到的 **symptom**，而不是 CPU / memory 這類 **cause**；dashboard 是「凍結的問題」，要定期修剪。

## 一個 event，三種視角

Event = **timestamp** + **context**（service、version、user、endpoint）+ **outcome**（成功 / 失敗、latency、error code）。

```
                         one event
  +------------------------------------------------------+
  | ts=12:00:03  service=payments  version=2.14          |
  | user_id=4127  endpoint=/checkout                     |
  | latency_ms=380  status=500                           |
  +------------------------------------------------------+
           |                  |                   |
           v                  v                   v
  +----------------+  +------------------+  +-------------------+
  | LOG            |  | METRIC           |  | TRACE             |
  | one record,    |  | failed_total +1  |  | one span in the   |
  | all fields     |  | 380 -> latency   |  | request tree:     |
  | kept           |  |   histogram      |  | gw -> pay -> db   |
  +----------------+  +------------------+  +-------------------+
   "what happened     "how often /          "where did this one
    this time?"        what trend?"          request go wrong?"
```

| | Log | Metric | Trace |
|---|---|---|---|
| 單位 | 一個 event | 大量 event 的聚合 | 一個 request 觸發的所有 event |
| 回答 | 這次發生了什麼？ | 多常發生？趨勢如何？ | 這個 request 在哪裡變慢 / 失敗？ |
| 串接靠 | 欄位（`request_id`、`trace_id`） | label | trace ID + parent span ID |
| 成本來源 | 量（volume） | **cardinality** | 量 → 需要 sampling |

## Logs

### Unstructured vs Structured

```
Unstructured  (opaque text)
  "User 4127 checkout failed after 380ms with status 500"
     -> grep "4127" ... and hope the substring is there

Structured  (named fields)
  {
    "ts": "2026-06-18T12:00:03Z",
    "level": "ERROR",
    "service": "payments",
    "endpoint": "/checkout",
    "user_id": 4127,
    "latency_ms": 380,
    "status": 500
  }
     -> WHERE user_id = 4127 AND status >= 500 AND ts > now() - 1h
```

- Unstructured：人讀起來自然，但每一行都是不透明的字串，「user 4127 過去一小時錯了幾次」只能靠字串比對。
- Structured：欄位有名字，就能像一般結構化資料一樣 **query、filter、aggregate**。

### Log levels

| Level | 用途 |
|---|---|
| `DEBUG` | 開發時的細節；排查時開、排查完關 |
| `INFO` | 正常的運作事件 |
| `WARN` | 還沒出錯，但值得知道 |
| `ERROR` | 真的失敗了 |

- Level 本質上是**執行期的 verbosity 開關**：不改 code 就能調整輸出量。
- 前提是團隊**用得一致**；每個人各自定義 ERROR，這個開關就失效。

### 限制

- 量大到一定程度，log 會變成淹沒自己訊號的噪音，儲存成本也跟著上來。
- **決定要 log 什麼**，比選哪個 logging library 更重要。

## Metrics

當問題從「這次發生了什麼」變成「整體發生得多頻繁」，就輪到 metrics。

| 類型 | 意義 | 例子 |
|---|---|---|
| Counter | 只增不減的累計值 | `http_requests_total` |
| Gauge | 當下的值，可升可降 | `queue_depth`、`memory_bytes` |
| Histogram | 數值分布（bucket） | `request_latency_ms` → p50 / p99 |

### Cardinality

Metric 可以帶 **dimension**（label / tag）。**每一種 label 值的組合，都是一條獨立的 time series。**

```
http_requests_total{ endpoint, pod, region, ... }

  endpoint   x 6                   ->           6 series
  + pod      x 50                  ->         300 series
  + region   x 10                  ->       3,000 series
  + user_id  x 1,000,000           ->   3,000,000,000 series   <-- boom
```

TSDB 的儲存方式是「每條 series、每個時間點存一個數值」：

```
series = metric name + one unique label combination

{endpoint="/checkout", status="500", region="ap"}  ->  t1:12   t2:15   t3:19  ...
{endpoint="/checkout", status="200", region="ap"}  ->  t1:880  t2:901  t3:940 ...
{endpoint="/login",    status="200", region="eu"}  ->  t1:310  t2:298  t3:305 ...
  ...
                 cost  ~  number of series   (index, memory)
                 cost !~  number of points written
```

- 成本跟著 **series 數量**走，不是寫入的點數。這是 TSDB 架構決定的，不是定價策略。
- 很多 observability 成本爆掉，追根究底都是某人加了一個「看起來無害」的 label。

| Low cardinality（有上限）→ 適合當 label | High cardinality（隨流量增長）→ 放 logs / traces |
|---|---|
| status code、region、environment、endpoint | user ID、session ID、request ID、order ID |

> 想「依客戶切 dashboard」很誘人，但 `user_id` 不該是 metric label。

## Traces

Metrics 描述所有 request 的整體狀況；trace 描述**某一個 request** 穿過多個服務時發生了什麼。

### Trace = span tree

每個 **span** 是一個工作單位（function call、對外 request、DB query），span 之間有 parent-child 關係。

```
trace_id = abc123                                   time ->
                                                    0ms            380ms
span A  api-gateway  GET /checkout                  [==============]
span B    payments   charge          parent=A        [============]
span C      db       SELECT orders   parent=B        [==]
span D      risk     POST /fraud     parent=B           [=======]
span E        risk   model.score     parent=D            [=====]
```

一眼就能看出：380ms 裡大部分卡在 risk service 的 `model.score`。

### Distributed context propagation

所有服務必須對同一個 request 使用**同一個 trace ID**，通常透過 request header 一路往下傳。

```
client          api-gateway            payments               risk
  |  GET /checkout   |                     |                     |
  |----------------->| new trace abc123    |                     |
  |                  | span A              |                     |
  |                  |  header: abc123/A   |                     |
  |                  |-------------------->| span B (parent=A)   |
  |                  |                     |  header: abc123/B   |
  |                  |                     |-------------------->| span D (parent=B)
  |                  |                     |                     |

  without propagation:  [A]   [B]   [D]   <- isolated, unrelated spans
  with propagation:     A -> B -> D       <- one tree per request
```

- 補充：標準格式是 W3C Trace Context 的 `traceparent: 00-<trace-id>-<parent-span-id>-<flags>`，`flags` 裡的 sampled bit 也讓下游知道要不要記錄。
- 前端角度：browser 也可以是 trace 的起點（OpenTelemetry Web 在 `fetch` 加上 `traceparent`）。跨域時後端的 `Access-Control-Allow-Headers` 要允許 `traceparent`，否則 preflight 會擋掉。

### Sampling

Request 量遠超過 tracing backend 能存的量，所以一定要挑。

```
HEAD SAMPLING                            TAIL SAMPLING
decide at request start                  decide after trace completes

  request                                  request
     |                                        |
     v                                        v
  hash(trace_id) < 10% ?                   every span buffered in collector
     |            |                           |
    yes           no                          v
     |            |                        trace complete -> look at its shape
   keep         drop                          |
     |                                        +-- error?        -> keep
  every service follows                       +-- slow?         -> keep
  the same decision                           +-- otherwise     -> keep random x%
```

| | Head sampling | Tail sampling |
|---|---|---|
| 決定時機 | request 開始時（還沒發生任何事） | trace 結束後（看得到全貌） |
| 依據 | 通常是 trace ID 本身 → 各服務決定一致 | error、latency、隨機抽樣 |
| 優點 | 便宜、簡單、一致 | 留得住有意義的 trace |
| 缺點 | 丟掉的可能正好是事後才知道重要的那筆 | 所有 span 都要 buffer，維運複雜、執行期成本高 |

## Monitoring vs Observability

```
+----------------------------------------------------------+
| OBSERVABILITY                                            |
| ask new questions nobody anticipated                     |
|   "why is it slow only for user X in region Y            |
|    since the last deploy?"                               |
|                                                          |
|   +--------------------------------------------------+   |
|   | MONITORING                                       |   |
|   | known checks on known systems, run continuously  |   |
|   |   - DB reachable?                                |   |
|   |   - error rate < 1% ?                            |   |
|   |   - queue depth in normal range?                 |   |
|   +--------------------------------------------------+   |
+----------------------------------------------------------+
```

- Monitoring 是 observability 資料能做到的事情的**子集**。
- 只有 observability、沒有 monitoring：某個三個月前就該觸發的例行檢查不見了，沒人發現。
- 只有 monitoring：碰到沒人預料到的怪問題，手上沒有能切的資料。

## Alerting

**3 am test**：凌晨三點被叫醒，responder 真的有事可做嗎？嚴重到值得叫醒人嗎？

```
                 PAGE  (symptom: what users feel)
  +-------------------------------------------------------+
  | checkout p99 latency | error rate on critical flows   |
  | request success rate on business-critical operations  |
  +-------------------------------------------------------+
                            |
                            |  responder starts here
                            v
                 INVESTIGATE  (cause: evidence, not pages)
  +-------------------------------------------------------+
  | CPU > 85%  |  memory > 80%  |  disk near full         |
  +-------------------------------------------------------+
```

- Cause 類指標可能**為真但使用者沒事**，也可能**為假但使用者已經在痛**。
- Cause 仍有價值，但定位是 symptom alert 把人叫來之後的**調查證據**。

**Alert fatigue 的連鎖**

```
page on causes -> false positives -> stop reading alerts carefully
   -> real alert missed -> slow incident response -> burnout / leave on-call
```

## Dashboards

Dashboard 是**凍結的問題**，不是資料展示牆。

```
BAD                                       GOOD
incident panic                            a question asked repeatedly
   |                                         |  "checkout latency vs last week?"
   v                                         v
30 panels "might be relevant"             1 panel answers it
   |                                         |
   v                                         v
nobody looks after 3 months               review on a regular cadence
   |                                         |
   v                                         v
wallpaper                                 prune panels nobody asks about
```

- 加 panel 前先問：**會有人反覆問這個問題嗎？** 「看起來有趣一次」不算。
- 系統會變，值得凍結的問題也會變；加 panel 很容易，**刪 panel 才需要紀律**。

## 延伸閱讀

- Google SRE Book, *Monitoring Distributed Systems*：four golden signals、symptom vs cause
- OpenTelemetry：logs / metrics / traces 的統一資料模型與 SDK
- W3C Trace Context：`traceparent` / `tracestate` header 規格
