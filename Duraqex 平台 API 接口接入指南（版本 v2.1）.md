# Duraqex 平台 API 接口接入指南（版本 v2.1）

**文档版本：** v2.1  
**行情模型：** 现货交易对行情  
**示例交易对：** `BTC-USDT`、`ETH-USDT`  
**时间基准：** UTC，Unix 毫秒时间戳  
**接口权限：** 本指南中的行情接口全部公开，只读，无须 API Key 或签名。

---

## 1. HTTP 行情接口

### 1.1 请求地址与接口列表

HTTP 基础地址：

```text
https://api.duraqex.com/v2.1
```

| 用途 | 请求方法 | 路径 | 数据范围 |
|---|---|---|---|
| 最新价格 | GET | `/market/ticker` | 单个交易对的最后一笔成交价格 |
| 盘口深度 | GET | `/market/depth` | 单个交易对当前买卖盘口快照 |
| 近期逐笔成交 | GET | `/market/trades` | 单个交易对最近的成交记录 |
| 历史及当前 K 线 | GET | `/market/klines` | 指定周期、时间范围的 K 线 |

所有查询参数放在 URL 查询字符串中，GET 请求不发送请求体。每次请求只允许指定一个交易对，不支持逗号分隔多个交易对。

请求示例：

```http
GET /v2.1/market/ticker?symbol=BTC-USDT HTTP/1.1
Host: api.duraqex.com
Accept: application/json
X-Request-ID: java-ticker-0001
```

`X-Request-ID` 可选，用于关联业务日志与服务端请求。仅允许英文字母、数字、下划线和连字符，长度为 1～64 个字符。合法时，服务端在响应体和同名响应头中原样返回；未传时由服务端生成。请求头非法时，服务端生成新的请求 ID，并返回参数错误。

### 1.2 通用字段与数据类型

| 类别 | 协议约定 | Java 处理方式 |
|---|---|---|
| 交易对 | 全大写，基础资产与计价资产以 `-` 分隔，例如 `BTC-USDT` | `String` |
| 时间戳 | JSON 整数，Unix 毫秒时间戳 | `long`、`Instant` |
| 价格 | JSON 字符串，例如 `"68250.10"` | `BigDecimal` |
| 基础资产数量 | JSON 字符串，例如 `"0.01500000"` | `BigDecimal` |
| 计价资产金额 | JSON 字符串，例如 `"1023.75150000"` | `BigDecimal` |
| 成交 ID、盘口序列号 | JSON 十进制数字字符串 | `String` 存储；需比较大小时按整数比较 |
| K 线修订号 | JSON 正整数 | `long` |
| 布尔值 | JSON `true` 或 `false` | `boolean` |
| 缺省游标、无可用数据字段 | JSON `null` | 使用可空类型，不能转成字符串 `"null"` |

补充约定：

1. 请求参数区分大小写。`btc-usdt`、`BTCUSDT` 和 `BTC-USDT` 不等价；只接受最后一种格式。
2. 不自动去除参数前后空格。参数名称重复、未知查询参数及无法解析的参数均返回 `INVALID_ARGUMENT`。
3. 时间戳必须是非负十进制整数，不接受秒级时间戳的自动转换，也不接受日期字符串。
4. 金额和数量不使用科学计数法；小数位数可能因交易对或字段不同而变化，不能根据示例固定小数位。
5. 数字字符串允许的 ID 和序列号范围为 `1`～`9223372036854775807`，无前导零；盘口的 `bookId` 不属于数字字符串。
6. 行情响应没有固定的小数位裁剪要求。Java 端应保留原始精度，仅在展示时按业务规则格式化。
7. 新增响应字段视为兼容变更，客户端应忽略未知字段。已有必需字段缺失、类型错误或未知消息类型必须记录并按对应数据流的恢复规则处理，不能静默当作正常数据。

### 1.3 统一响应结构

成功响应：

```json
{
  "code": "0",
  "message": "OK",
  "requestId": "java-ticker-0001",
  "serverTime": 1792656072000,
  "data": {
    "symbol": "BTC-USDT"
  }
}
```

上例仅展示响应外层，完整业务字段见各接口。

| 字段 | 类型 | 含义 |
|---|---|---|
| `code` | string | 成功固定为 `"0"`，失败为错误码字符串 |
| `message` | string | 可读说明，不作为程序分支判断依据 |
| `requestId` | string | 本次请求的关联 ID |
| `serverTime` | integer | 生成响应时的服务端时间，毫秒 |
| `data` | object / null | 成功时为业务对象，失败时为 `null` |
| `error` | object | 仅失败时出现，包含结构化错误详情 |

HTTP 状态码和业务 `code` 必须同时判断。HTTP `200` 对应 `code="0"`；业务错误使用对应的非 2xx 状态码。代理或网关产生的错误可能没有上述 JSON 外层，应允许读取非 JSON 错误正文。

错误响应示例，HTTP 状态码为 `400`：

```json
{
  "code": "INVALID_ARGUMENT",
  "message": "limit must be between 1 and 1000",
  "requestId": "java-kline-0002",
  "serverTime": 1792656072000,
  "data": null,
  "error": {
    "details": [
      {
        "field": "limit",
        "reason": "OUT_OF_RANGE"
      }
    ]
  }
}
```

### 1.4 最新价格

#### 1.4.1 接口定义

```http
GET /v2.1/market/ticker
```

获取指定交易对最后一笔成交形成的价格。该价格不是盘口中间价，也不代表现在一定能够按该价格成交。

#### 1.4.2 请求参数

| 参数 | 类型 | 必填 | 默认值 | 说明 |
|---|---|---|---|---|
| `symbol` | string | 是 | 无 | 交易对，例如 `BTC-USDT` |

```text
https://api.duraqex.com/v2.1/market/ticker?symbol=BTC-USDT
```

#### 1.4.3 响应示例

```json
{
  "code": "0",
  "message": "OK",
  "requestId": "java-ticker-0001",
  "serverTime": 1792656072000,
  "data": {
    "symbol": "BTC-USDT",
    "hasTrade": true,
    "lastPrice": "68250.10",
    "lastQty": "0.01500000",
    "lastTradeId": "90010002",
    "lastTradeTime": 1792656071950
  }
}
```

#### 1.4.4 字段说明

| 字段 | 类型 | 可空 | 说明 |
|---|---|---|---|
| `symbol` | string | 否 | 交易对 |
| `hasTrade` | boolean | 否 | 该交易对是否已有成交 |
| `lastPrice` | string | 是 | 最后成交价，单位为 USDT/BTC |
| `lastQty` | string | 是 | 最后一笔成交的基础资产数量，示例中单位为 BTC |
| `lastTradeId` | string | 是 | 最后一笔成交 ID |
| `lastTradeTime` | integer | 是 | 最后一笔成交发生时间 |

对于已存在但从未成交的交易对，接口仍返回成功，`hasTrade=false`，其余四个成交字段均为 `null`。没有新成交时，成交字段保持不变，不能用 HTTP `serverTime` 代替成交时间判断行情新旧。

最新价格响应不包含 24 小时涨跌幅、24 小时成交量等统计数据，客户端不应将单笔数量 `lastQty` 当作累计成交量。

### 1.5 盘口深度

#### 1.5.1 接口定义

```http
GET /v2.1/market/depth
```

返回同一盘口状态下生成的买卖两侧快照。HTTP 快照可用于一次性展示、重查或校验；实时盘口初始化优先使用 WebSocket 订阅后推送的快照，具体流程见第 2.6 节。

#### 1.5.2 请求参数

| 参数 | 类型 | 必填 | 默认值 | 说明 |
|---|---|---|---|---|
| `symbol` | string | 是 | 无 | 交易对 |
| `limit` | integer | 否 | `20` | 每一侧的档位上限，只允许 `5、10、20、50、100、500` |

`limit=5` 表示最多返回五档买价和五档卖价，不是两侧合计五档。

```text
https://api.duraqex.com/v2.1/market/depth?symbol=BTC-USDT&limit=5
```

#### 1.5.3 响应示例

```json
{
  "code": "0",
  "message": "OK",
  "requestId": "java-depth-0001",
  "serverTime": 1792656072000,
  "data": {
    "symbol": "BTC-USDT",
    "limit": 5,
    "bookId": "dep-btc-usdt-0007",
    "sequence": "8000100",
    "eventTime": 1792656071980,
    "bids": [
      ["68250.00", "1.20000000"],
      ["68249.90", "0.80000000"],
      ["68249.80", "2.50000000"],
      ["68249.70", "1.10000000"],
      ["68249.60", "0.60000000"]
    ],
    "asks": [
      ["68250.20", "0.90000000"],
      ["68250.30", "1.40000000"],
      ["68250.40", "2.10000000"],
      ["68250.50", "0.70000000"],
      ["68250.60", "1.80000000"]
    ]
  }
}
```

#### 1.5.4 字段及排序规则

| 字段 | 类型 | 说明 |
|---|---|---|
| `symbol` | string | 交易对 |
| `limit` | integer | 请求的每侧档位上限 |
| `bookId` | string | 盘口序列所属的生命周期标识；盘口重建时变化 |
| `sequence` | string | 该快照的盘口状态序列号，在同一 `bookId` 内递增 |
| `eventTime` | integer | 此盘口状态形成的时间 |
| `bids` | array | 买盘，按价格从高到低排列 |
| `asks` | array | 卖盘，按价格从低到高排列 |

每个档位严格为二元数组 `[price, quantity]`：

- `price`：价格，示例中为 USDT/BTC。
- `quantity`：该价格档位聚合的剩余基础资产数量，示例中为 BTC。
- 同一侧每个价格只出现一次，快照中不包含数量为零的档位。
- 价格和数量均为正数；某侧没有挂单时返回空数组 `[]`。
- 实际档位不足 `limit` 时按实际数量返回，不补零，不保证两侧档位数相等。
- 同一 `bookId` 的 HTTP 与 WebSocket `sequence` 使用同一来源序列；不同 `bookId` 的序列不可比较。

最优买价是非空 `bids` 的第一项，最优卖价是非空 `asks` 的第一项。盘口买卖价差可由最优卖价减最优买价得到，但不能用价差反推最新成交价。

### 1.6 近期逐笔成交

#### 1.6.1 接口定义

```http
GET /v2.1/market/trades
```

获取最近若干笔实际撮合成交。每一项是一笔成交，不是订单，也不是按价格聚合后的统计。

#### 1.6.2 请求参数

| 参数 | 类型 | 必填 | 默认值 | 说明 |
|---|---|---|---|---|
| `symbol` | string | 是 | 无 | 交易对 |
| `limit` | integer | 否 | `100` | 返回笔数上限，范围 `1`～`1000` |

```text
https://api.duraqex.com/v2.1/market/trades?symbol=BTC-USDT&limit=2
```

该接口不支持按时间范围、成交 ID 或游标查询。它只提供最近最多 1000 笔成交，不能承担任意历史逐笔数据下载。

#### 1.6.3 响应示例

```json
{
  "code": "0",
  "message": "OK",
  "requestId": "java-trades-0001",
  "serverTime": 1792656072000,
  "data": {
    "symbol": "BTC-USDT",
    "limit": 2,
    "count": 2,
    "items": [
      {
        "tradeId": "90010001",
        "price": "68249.90",
        "quantity": "0.02000000",
        "quoteQuantity": "1364.99800000",
        "takerSide": "SELL",
        "tradeTime": 1792656071870
      },
      {
        "tradeId": "90010002",
        "price": "68250.10",
        "quantity": "0.01500000",
        "quoteQuantity": "1023.75150000",
        "takerSide": "BUY",
        "tradeTime": 1792656071950
      }
    ]
  }
}
```

#### 1.6.4 字段说明

| 字段 | 类型 | 说明 |
|---|---|---|
| `data.symbol` | string | 交易对 |
| `data.limit` | integer | 请求笔数上限 |
| `data.count` | integer | 本次实际返回笔数，等于 `items.length` |
| `tradeId` | string | 交易对内唯一、按成交顺序递增的成交 ID |
| `price` | string | 成交价格 |
| `quantity` | string | 基础资产成交数量 |
| `quoteQuantity` | string | 计价资产成交金额；本协议为 `price × quantity` 的精确十进制结果，不含手续费 |
| `takerSide` | string | 主动成交方向：`BUY` 或 `SELL` |
| `tradeTime` | integer | 成交发生时间 |

方向解释：

- `BUY`：主动买方与盘口卖单成交。
- `SELL`：主动卖方与盘口买单成交。
- 该字段不是当前接入方的买卖方向，也不表示只有买方或只有卖方参与成交。

排序与去重规则：

1. 先选取最新的 `limit` 笔，再按 `tradeId` 数值升序返回，便于顺序处理。
2. 同一个毫秒内可以发生多笔成交，不能只用 `tradeTime` 去重。
3. 去重键为 `(symbol, tradeId)`。
4. 成交 ID 单调递增但不保证连续；ID 跳号本身不能证明丢单。
5. 没有成交记录时，`count=0`，`items=[]`，HTTP 仍为 `200`。

### 1.7 历史及当前 K 线

#### 1.7.1 接口定义

```http
GET /v2.1/market/klines
```

返回按指定周期聚合的 K 线。允许同时返回已收盘 K 线和当前未收盘 K 线，是否收盘必须读取 `isClosed`。

#### 1.7.2 请求参数

| 参数 | 类型 | 必填 | 默认值 | 说明 |
|---|---|---|---|---|
| `symbol` | string | 是 | 无 | 交易对 |
| `interval` | string | 是 | 无 | `1m、5m、15m、30m、1h、4h、1d` 之一 |
| `startTime` | integer | 否 | 见分页规则 | 开盘时间下界，包含该值 |
| `endTime` | integer | 否 | 见分页规则 | 开盘时间上界，不包含该值 |
| `limit` | integer | 否 | `500` | 返回条数上限，范围 `1`～`1000` |

请求示例：

```text
https://api.duraqex.com/v2.1/market/klines?symbol=BTC-USDT&interval=1m&startTime=1792656000000&endTime=1792656120000&limit=2
```

这个请求筛选 UTC `2026-10-22 08:00:00` 至 `08:02:00` 之间开盘的 1 分钟 K 线，包含 `08:00:00` 和 `08:01:00`，不包含 `08:02:00`。

#### 1.7.3 响应示例

```json
{
  "code": "0",
  "message": "OK",
  "requestId": "java-kline-0001",
  "serverTime": 1792656072000,
  "data": {
    "symbol": "BTC-USDT",
    "interval": "1m",
    "intervalMs": 60000,
    "asOf": 1792656072000,
    "availableFrom": 1704067200000,
    "pageDirection": "forward",
    "limit": 2,
    "count": 2,
    "hasMore": false,
    "nextStartTime": null,
    "nextEndTime": null,
    "items": [
      {
        "openTime": 1792656000000,
        "closeTime": 1792656060000,
        "open": "68200.00",
        "high": "68300.00",
        "low": "68180.00",
        "close": "68240.00",
        "volume": "12.00000000",
        "quoteVolume": "818760.00000000",
        "tradeCount": 128,
        "isClosed": true,
        "revision": 129,
        "eventTime": 1792656060000
      },
      {
        "openTime": 1792656060000,
        "closeTime": 1792656120000,
        "open": "68240.00",
        "high": "68260.00",
        "low": "68230.00",
        "close": "68250.10",
        "volume": "2.50000000",
        "quoteVolume": "170610.12500000",
        "tradeCount": 31,
        "isClosed": false,
        "revision": 31,
        "eventTime": 1792656071950
      }
    ]
  }
}
```

本例 `availableFrom=1704067200000` 代表该交易对及周期在示例服务中最早可查询的 K 线开盘时间，即 UTC `2024-01-01 00:00:00`。真实接入不得将此示例值写死。

#### 1.7.4 分页元数据

| 字段 | 类型 | 说明 |
|---|---|---|
| `symbol` | string | 交易对 |
| `interval` | string | 周期字符串 |
| `intervalMs` | integer | 当前周期对应的毫秒数 |
| `asOf` | integer | 本次查询读取数据快照的时间，不晚于外层 `serverTime` |
| `availableFrom` | integer / null | 最早可查询 K 线的开盘时间；尚无任何 K 线时为 `null` |
| `pageDirection` | string | `forward` 或 `backward`，决定下一页游标 |
| `limit` | integer | 请求条数上限 |
| `count` | integer | 返回条数，等于 `items.length` |
| `hasMore` | boolean | 在本次 `asOf` 已存在、且满足原筛选范围的数据中，该方向是否还有未返回记录 |
| `nextStartTime` | integer / null | 向前翻页时使用的下一页下界，仅 `forward` 且 `hasMore=true` 时非空 |
| `nextEndTime` | integer / null | 向后翻页时使用的下一页上界，仅 `backward` 且 `hasMore=true` 时非空 |
| `items` | array | K 线数组，所有模式下均按 `openTime` 升序排列 |

#### 1.7.5 单根 K 线字段

| 字段 | 类型 | 说明 |
|---|---|---|
| `openTime` | integer | 周期开始时间，包含该时刻 |
| `closeTime` | integer | 周期结束边界，不包含该时刻；不是“最后一毫秒” |
| `open` | string | 本周期第一笔成交价格；无成交周期的特殊规则见第 3.4 节 |
| `high` | string | 本周期最高成交价 |
| `low` | string | 本周期最低成交价 |
| `close` | string | 本周期最后成交价；未收盘时为截至当前的最后成交价 |
| `volume` | string | 本周期累计基础资产成交量 |
| `quoteVolume` | string | 本周期累计计价资产成交额，即逐笔 `price × quantity` 求和 |
| `tradeCount` | integer | 本周期成交笔数，非负，Java 使用 `long` |
| `isClosed` | boolean | 服务端是否已完成本周期收盘 |
| `revision` | integer | 同一根 K 线的修订号，初始为 `1`，业务字段更新时递增，不保证连续推送 |
| `eventTime` | integer | 这一修订版本形成的时间 |

同一根 K 线由 `(symbol, interval, openTime)` 唯一确定。HTTP 与 WebSocket 共用这组键和同一套 `revision`，用于初始化和实时数据合并。

### 1.8 HTTP 限流、超时与错误处理

本设计按来源 IP 对四个 HTTP 行情接口合并计费。固定 UTC 自然分钟窗口内最多使用 **600 个权重单位**。

| 接口 | 单次请求权重 |
|---|---:|
| `/market/ticker` | 1 |
| `/market/depth`，`limit≤100` | 2 |
| `/market/depth`，`limit=500` | 5 |
| `/market/trades` | 2 |
| `/market/klines` | 5 |

经过基础路由后、参数合法的查询请求会消耗权重，包括没有匹配数据的成功请求和进入查询后发生的服务错误；参数校验失败不扣减上述行情额度。被限流拒绝的请求不继续扣减额度。

服务端业务响应携带：

| 响应头 | 说明 |
|---|---|
| `X-RateLimit-Limit` | 当前窗口总额度，例如 `600` |
| `X-RateLimit-Remaining` | 当前响应对应请求计费后剩余额度 |
| `X-RateLimit-Reset` | 当前窗口结束时间，Unix 毫秒时间戳 |
| `Retry-After` | 仅 HTTP `429` 时必有，值为至少应等待的整数秒数 |

限流窗口按服务端时间计算。并发请求的响应到达顺序可能与计费顺序不同，不能把任意一个响应中的剩余额度当成全局实时锁。

| HTTP 状态 | `code` | 含义 | Java 客户端处理 |
|---:|---|---|---|
| 400 | `INVALID_ARGUMENT` | 参数类型、范围、格式或重复参数错误 | 修改参数，不自动原样重试 |
| 400 | `INVALID_INTERVAL` | 不支持的 K 线周期 | 按周期枚举修正 |
| 400 | `INVALID_TIME_RANGE` | 同时传入的 `startTime≥endTime` | 修正范围 |
| 404 | `SYMBOL_NOT_FOUND` | 交易对不存在 | 核对交易对 |
| 404 | `NOT_FOUND` | 路径不存在 | 核对版本和路径 |
| 405 | `METHOD_NOT_ALLOWED` | 使用非 GET 方法 | 改为 GET；响应头 `Allow: GET` |
| 429 | `RATE_LIMITED` | 行情权重额度不足 | 至少等待 `Retry-After` 指定时长 |
| 500 | `INTERNAL_ERROR` | 服务内部异常 | 有上限地退避重试 |
| 503 | `SERVICE_UNAVAILABLE` | 当前行情服务不可用 | 有上限地退避重试，数据标记为过期 |

返回空数组、当前没有成交、盘口某侧为空均属于正常业务状态，不使用 `404` 或 `204` 表示。

Java 调用可将连接超时设为 5 秒、单次请求超时设为 10 秒作为初始配置，再按部署情况调整。这些值是客户端配置建议，不是服务端延迟承诺。GET 请求可在网络瞬断或 `5xx` 时重试，但需设置最大尝试次数；`429` 必须优先遵守 `Retry-After`。

---

## 2. WebSocket 行情接入

### 2.1 连接地址与消息约定

```text
wss://stream.duraqex.com/v2.1/market
```

本地址承载最新价格、盘口深度、逐笔成交和实时 K 线四类频道。连接本身不自动订阅任何行情；连接成功后需要发送订阅请求，并等待确认。

| 项目 | 约定 |
|---|---|
| 传输协议 | TLS 上的 WebSocket，即 `wss` |
| 应用消息 | UTF-8 文本，每个完整 WebSocket 消息包含一个 JSON 对象 |
| 二进制消息 | 本版本不使用 |
| 应用层压缩 | 不发送需要客户端自行 gzip 解压的业务数据 |
| 子协议 | 无须设置 `Sec-WebSocket-Protocol` |
| 单条应用消息上限 | 双向均为解码后的 UTF-8 文本 1 MiB |
| 公共认证 | 无须 API Key、签名或登录消息 |
| 单连接订阅上限 | 50 个规范化订阅 |
| 单 IP 并发连接上限 | 10 个；超过时在握手阶段返回 HTTP `429` |
| 客户端控制消息频率 | 每连接任意连续 1 秒内不超过 10 条，含订阅、取消订阅、应用层 ping |
| 一次订阅请求 | 最多 20 个订阅项 |
| 服务端历史重放 | 不提供断线期间消息自动重放，也不支持携带断点恢复连接 |

WebSocket 消息可能分成多个传输片段。Java 回调收到的单次文本不一定就是完整 JSON；应累积到完整消息结束后再解析。

服务端对同一连接按发送顺序交付消息。不同频道之间不提供原子一致性，例如价格频道的成交价可能先于同一笔成交引起的盘口更新到达。不要通过跨频道的到达先后推断撮合过程。

### 2.2 订阅标识

| 频道 | `channel` | 必填订阅字段 | 可选字段 | 规范化唯一键 |
|---|---|---|---|---|
| 最新价格 | `ticker` | `symbol` | 无 | `(ticker, symbol)` |
| 盘口深度 | `depth` | `symbol` | `limit`，默认 `20` | `(depth, symbol, limit)` |
| 逐笔成交 | `trades` | `symbol` | 无 | `(trades, symbol)` |
| 实时 K 线 | `kline` | `symbol, interval` | 无 | `(kline, symbol, interval)` |

WebSocket 盘口 `limit` 只支持 `5、10、20、50、100`，不支持 HTTP 的 `500` 档。同一连接、同一交易对只能保留一个盘口订阅深度；如需从 20 档切换为 100 档，应先取消 20 档并等待确认，再订阅 100 档。

同一交易对可同时订阅多个 K 线周期，每个周期分别占用一个订阅名额。四类频道各订阅一次共占用四个名额。

`requestId` 在每条客户端控制消息中必填，格式与 HTTP 请求 ID 一致。它只关联控制请求与控制响应，不用于行情去重。客户端应为每次控制操作生成新的 ID；本协议不提供控制请求 ID 重放缓存。

### 2.3 订阅与取消订阅

#### 2.3.1 一次订阅四类行情

客户端发送：

```json
{
  "op": "subscribe",
  "requestId": "sub-0001",
  "args": [
    {
      "channel": "ticker",
      "symbol": "BTC-USDT"
    },
    {
      "channel": "depth",
      "symbol": "BTC-USDT",
      "limit": 5
    },
    {
      "channel": "trades",
      "symbol": "BTC-USDT"
    },
    {
      "channel": "kline",
      "symbol": "BTC-USDT",
      "interval": "1m"
    }
  ]
}
```

服务端确认：

```json
{
  "op": "subscribe",
  "requestId": "sub-0001",
  "success": true,
  "ts": 1792656072000,
  "data": {
    "subscriptions": [
      {
        "channel": "ticker",
        "symbol": "BTC-USDT"
      },
      {
        "channel": "depth",
        "symbol": "BTC-USDT",
        "limit": 5
      },
      {
        "channel": "trades",
        "symbol": "BTC-USDT"
      },
      {
        "channel": "kline",
        "symbol": "BTC-USDT",
        "interval": "1m"
      }
    ],
    "totalActive": 4
  }
}
```

订阅处理规则：

1. `args` 必须为非空数组，长度不超过 20。
2. 订阅请求中存在非法字段、非法周期、未知交易对、深度冲突或超过订阅上限时，整批失败，原有订阅保持不变，不采用部分成功模式。
3. 同一请求中重复的规范化订阅项只计一次。
4. 已存在的相同订阅再次订阅会成功，不重复计数，不重复发送初始快照。
5. `data.subscriptions` 返回本次成功处理、去重并补齐默认值后的订阅项，包含已经存在的项；`totalActive` 为该连接当前有效订阅总数。
6. 对新建立的订阅，确认消息先发送，随后才发送该订阅的数据。其他原有订阅的数据可以正常穿插。
7. 初始行为：`ticker` 发送最新状态快照，`depth` 发送完整档位快照，`kline` 发送当前周期状态快照，`trades` 仅推送订阅生效后产生的新成交。

#### 2.3.2 取消订阅

客户端发送：

```json
{
  "op": "unsubscribe",
  "requestId": "unsub-0001",
  "args": [
    {
      "channel": "depth",
      "symbol": "BTC-USDT",
      "limit": 5
    }
  ]
}
```

服务端确认：

```json
{
  "op": "unsubscribe",
  "requestId": "unsub-0001",
  "success": true,
  "ts": 1792656075000,
  "data": {
    "subscriptions": [
      {
        "channel": "depth",
        "symbol": "BTC-USDT",
        "limit": 5
      }
    ],
    "totalActive": 3
  }
}
```

取消订阅使用与订阅相同的参数校验和批量数量限制，并且整批原子处理。取消一个格式合法、交易对存在但当前未订阅的项目，也返回成功。

盘口取消请求必须匹配规范化 `limit`：省略时按 20 档处理，因此不能通过省略 `limit` 来取消 5 档订阅。

确认消息之前可能还有已进入发送队列的旧数据；确认消息之后，服务端不再发送该项订阅的数据。客户端自己的异步处理队列也应保留订阅代次，防止已取消订阅的旧任务覆盖新订阅状态。

#### 2.3.3 订阅失败

```json
{
  "op": "subscribe",
  "requestId": "sub-0002",
  "success": false,
  "ts": 1792656072000,
  "error": {
    "code": "INVALID_INTERVAL",
    "message": "unsupported interval: 2m",
    "details": [
      {
        "field": "args[0].interval",
        "reason": "UNSUPPORTED_VALUE"
      }
    ]
  }
}
```

控制响应以 `op` 识别，业务数据以 `channel` 识别。解析时先区分这两类消息，不能把订阅确认或心跳响应作为行情对象处理。

### 2.4 行情消息公共结构

| 字段 | 类型 | 说明 |
|---|---|---|
| `channel` | string | `ticker`、`depth`、`trades` 或 `kline` |
| `symbol` | string | 交易对 |
| `type` | string | `snapshot` 或 `update` |
| `ts` | integer | 服务端将该消息交给发送队列的时间 |
| `data` | object / array / null | 频道业务载荷 |
| `limit` | integer | 仅盘口频道存在，表示订阅档位数 |
| `interval` | string | 仅 K 线频道存在，表示订阅周期 |

`ts` 不等于交易发生时间，也不等于客户端收到时间。需要业务时间时，读取 `lastTradeTime`、`tradeTime` 或 `eventTime` 等频道字段。

本协议不承诺跨重连的恰好一次交付。客户端应允许快照和 HTTP 补查造成的业务数据重复；断线期间可能缺失数据，需要按频道恢复。

下列频率是虚构协议中的正常推送节奏，用于说明聚合方式，不是实际服务性能承诺：

| 频道 | 正常推送节奏 | 是否可能合并中间状态 |
|---|---|---|
| `ticker` | 最短约 100 ms，有新成交才更新 | 会，只保留窗口内最新成交状态 |
| `depth` | 最短约 100 ms，所订阅档位状态变化才更新 | 会，多次变化合并为最终档位数量 |
| `trades` | 约 50 ms 组批，每条消息最多 100 笔 | 不合并不同成交；超过 100 笔拆成多条消息 |
| `kline` | 常规更新约 1000 ms，收盘及新周期开启另行发送 | 会，但每次都是该 K 线的完整累计状态 |

上述节奏不意味着空闲市场也会定时推送。连接存活依靠心跳判断。

### 2.5 最新价格频道

#### 2.5.1 订阅参数

```json
{
  "op": "subscribe",
  "requestId": "sub-ticker-0001",
  "args": [
    {
      "channel": "ticker",
      "symbol": "BTC-USDT"
    }
  ]
}
```

#### 2.5.2 推送示例

首次为 `type="snapshot"`；后续更新示例：

```json
{
  "channel": "ticker",
  "symbol": "BTC-USDT",
  "type": "update",
  "ts": 1792656072000,
  "data": {
    "hasTrade": true,
    "lastPrice": "68250.10",
    "lastQty": "0.01500000",
    "lastTradeId": "90010002",
    "lastTradeTime": 1792656071950
  }
}
```

字段含义与 HTTP 最新价格相同，`symbol` 位于消息外层，不在 `data` 中重复。

该频道的 `snapshot` 与 `update` 都是完整状态，直接替换该交易对已有状态即可。在 HTTP 初始化与 WebSocket 实时数据竞争时，应按 `lastTradeId` 数值比较保留较新的记录；相同 ID 视为重复。若本地已有成交，不得用较早获取的 `hasTrade=false` 快照覆盖。

没有成交历史时，首次快照仍会发送，`hasTrade=false`，四个成交字段为 `null`。由于该频道会合并中间价格变化，需要逐笔记录时必须使用 `trades` 频道。

### 2.6 盘口深度频道

#### 2.6.1 初始快照

新订阅确认后，先接收 `snapshot`：

```json
{
  "channel": "depth",
  "symbol": "BTC-USDT",
  "limit": 5,
  "type": "snapshot",
  "ts": 1792656072000,
  "data": {
    "bookId": "dep-btc-usdt-0007",
    "sequence": "8000100",
    "eventTime": 1792656071980,
    "bids": [
      ["68250.00", "1.20000000"],
      ["68249.90", "0.80000000"],
      ["68249.80", "2.50000000"],
      ["68249.70", "1.10000000"],
      ["68249.60", "0.60000000"]
    ],
    "asks": [
      ["68250.20", "0.90000000"],
      ["68250.30", "1.40000000"],
      ["68250.40", "2.10000000"],
      ["68250.50", "0.70000000"],
      ["68250.60", "1.80000000"]
    ]
  }
}
```

收到快照后，整体替换本地买卖盘口，记录 `bookId`、`sequence` 和当前订阅代次，再将盘口状态标记为可用。

快照表示该时刻前 N 档的完整集合，不包含 N 档之外的盘口。不得把不同订阅深度或不同 `bookId` 的快照与增量混在一起。

#### 2.6.2 增量更新

```json
{
  "channel": "depth",
  "symbol": "BTC-USDT",
  "limit": 5,
  "type": "update",
  "ts": 1792656072100,
  "data": {
    "bookId": "dep-btc-usdt-0007",
    "prevSequence": "8000100",
    "sequence": "8000107",
    "eventTime": 1792656072080,
    "bids": [
      ["68250.00", "0.95000000"],
      ["68249.90", "0.00000000"],
      ["68249.50", "1.30000000"]
    ],
    "asks": [
      ["68250.20", "1.10000000"]
    ]
  }
}
```

| 字段 | 说明 |
|---|---|
| `bookId` | 必须与当前盘口生命周期一致 |
| `prevSequence` | 此订阅前一次已发送盘口状态的序列号，必须等于本地已应用序列号 |
| `sequence` | 应用本次更新后的序列号，必须大于 `prevSequence` |
| `eventTime` | 本次目标盘口状态形成时间 |
| `bids`、`asks` | 相对于上一条已发送状态发生变化的档位；未变化的一侧可为空数组 |

**增量中的数量是该价位的新绝对数量，不是加减差值。** 示例中的 `"0.95000000"` 表示将 `68250.00` 档位替换为 `0.95 BTC`，不是在原来 `1.2 BTC` 上增加 `0.95 BTC`。

更新规则：

1. 数量大于零：新增或替换该价格档位。
2. 数量为零：删除该价格档位，不论本地是否存在。
3. 数量小于零：非法消息，停止应用并重建盘口。
4. 同一条消息中的两侧变化作为一个整体应用，完成后再发布给业务线程。
5. 档位价格不得重复，买侧增量按价格降序排列，卖侧增量按价格升序排列。

应用示例增量后，本地五档买盘应为：

```text
68250.00  ->  0.95000000
68249.80  ->  2.50000000
68249.70  ->  1.10000000
68249.60  ->  0.60000000
68249.50  ->  1.30000000
```

#### 2.6.3 前 N 档边界处理

本协议发送的是“前 N 档集合之间的差异”，不是只发送交易引擎中实际撤单的档位：

- 原来在前 N 档、更新后移出前 N 档的价位，发送数量为零的删除项，即使该价位仍存在于更深盘口。
- 新进入前 N 档的价位，发送其完整当前数量，即使该价位只是从第 N+1 档补入。
- 留在前 N 档但数量变化的价位，发送新数量。
- 前 N 档之外且未影响当前前 N 档集合的变化，可以不推送。

因此，客户端按上述规则应用完整消息后，应得到当前前 N 档的精确集合。不要仅依靠“截断到 N 档”来掩盖漏删或漏补问题；若结果违反协议，应重建盘口。

#### 2.6.4 序列连续性校验

对同一订阅代次，按以下顺序处理：

1. 没有可用快照时，不应用 `update`。
2. `bookId` 与本地不一致时，不比较序列大小，不应用此增量，等待或重新请求新快照。
3. 相同 `bookId` 且 `sequence≤localSequence`：属于重复或过期状态，忽略。
4. `sequence>localSequence` 且 `prevSequence=localSequence`：正常应用。
5. `sequence>localSequence` 但 `prevSequence≠localSequence`：存在缺口、乱序或错误合并，盘口立即标记为不可用，执行重新订阅。

不要求 `sequence=prevSequence+1`。示例由 `8000100` 更新到 `8000107` 是合法的，因为服务端可能合并了多次底层变化。

服务端盘口重建时，可以主动推送新的 `snapshot`，其 `bookId` 可能变化。新快照会替换旧状态，后续增量从该快照继续；正常协议顺序不会先发送新 `bookId` 的增量再发送对应快照。当前订阅内相同 `bookId`、不新于本地序列的重复快照可以忽略。

#### 2.6.5 发现缺口后的恢复

1. 将该交易对盘口标记为 `STALE`，停止将其作为实时有效盘口提供给业务。
2. 暂停应用该订阅后续增量。
3. 发送取消订阅，等待确认。
4. 重新订阅相同 `symbol` 与 `limit`，等待新的订阅确认和快照。
5. 用新快照替换全部本地状态，再恢复增量处理。

若取消或重新订阅在 5 秒内未确认，关闭连接并执行连接重建；共享连接上的其他频道也要恢复订阅。

不能只重复发送同一个 `subscribe` 期待获得新快照，因为已经存在的订阅是幂等成功，不会重新发送快照。

不要直接把任意 HTTP 快照插入正在运行的 WebSocket 增量链：HTTP 读取时刻可能位于两条已合并增量之间，无法保证下一条增量的 `prevSequence` 正好匹配。此版本将“重新订阅后取得 WebSocket 快照”定义为标准恢复流程。

### 2.7 逐笔成交频道

#### 2.7.1 推送示例

```json
{
  "channel": "trades",
  "symbol": "BTC-USDT",
  "type": "update",
  "ts": 1792656072000,
  "data": [
    {
      "tradeId": "90010001",
      "price": "68249.90",
      "quantity": "0.02000000",
      "quoteQuantity": "1364.99800000",
      "takerSide": "SELL",
      "tradeTime": 1792656071870
    },
    {
      "tradeId": "90010002",
      "price": "68250.10",
      "quantity": "0.01500000",
      "quoteQuantity": "1023.75150000",
      "takerSide": "BUY",
      "tradeTime": 1792656071950
    }
  ]
}
```

`data` 是长度为 1～100 的数组。即使只有一笔成交也保持数组，不发送空数组。字段与 HTTP 成交记录完全一致。

同一消息内、以及同一持续连接的该交易对数据流内，按 `tradeId` 数值升序交付。不同交易对的 ID 不互相比较。

#### 2.7.2 初始化与断线补查

需要“近期成交列表 + 实时更新”时：

1. 先订阅 `trades`，确认成功后开始缓存实时成交。
2. 请求 HTTP `/market/trades` 获取最近记录。
3. 将 HTTP 结果与缓存数据按 `(symbol, tradeId)` 合并、去重。
4. 按 `tradeId` 数值升序处理，然后继续消费实时数据。

断线重连时采用相同方式，但是否能够完整补齐取决于最近 1000 笔的覆盖范围：

- HTTP 返回中包含断线前已经处理过的最后一笔成交 ID，且新实时流衔接正常时，可建立补查与本地已有记录的重叠边界。
- 若断线前最后一笔成交早于 HTTP 窗口最早记录，不能确认缺失成交已全部补齐。
- HTTP 查询和实时缓冲切换期间也需要检查重叠或明确的订阅生效边界，不能仅靠比较时间戳认定无缺口。
- 必须完整保存所有历史逐笔数据的系统，应持续落库，并显式记录无法补齐的缺口；本协议不提供更早逐笔成交的分页接口。

不能通过 `ticker` 补回所有逐笔成交，因为最新价格频道会合并中间状态。

### 2.8 实时 K 线频道

#### 2.8.1 订阅请求

```json
{
  "op": "subscribe",
  "requestId": "sub-kline-0001",
  "args": [
    {
      "channel": "kline",
      "symbol": "BTC-USDT",
      "interval": "1m"
    },
    {
      "channel": "kline",
      "symbol": "BTC-USDT",
      "interval": "1h"
    }
  ]
}
```

#### 2.8.2 当前 K 线更新

```json
{
  "channel": "kline",
  "symbol": "BTC-USDT",
  "interval": "1m",
  "type": "update",
  "ts": 1792656072000,
  "data": {
    "openTime": 1792656060000,
    "closeTime": 1792656120000,
    "open": "68240.00",
    "high": "68260.00",
    "low": "68230.00",
    "close": "68250.10",
    "volume": "2.50000000",
    "quoteVolume": "170610.12500000",
    "tradeCount": 31,
    "isClosed": false,
    "revision": 31,
    "eventTime": 1792656071950
  }
}
```

初始当前周期快照使用 `type="snapshot"`，载荷与更新相同；如果交易对尚未出现第一笔成交、当前周期还没有 K 线，则初始快照的 `data=null`。这不代表订阅失败，后续第一根 K 线形成时仍会正常推送。

#### 2.8.3 收盘消息

```json
{
  "channel": "kline",
  "symbol": "BTC-USDT",
  "interval": "1m",
  "type": "update",
  "ts": 1792656120050,
  "data": {
    "openTime": 1792656060000,
    "closeTime": 1792656120000,
    "open": "68240.00",
    "high": "68280.00",
    "low": "68230.00",
    "close": "68270.00",
    "volume": "8.00000000",
    "quoteVolume": "546040.00000000",
    "tradeCount": 96,
    "isClosed": true,
    "revision": 97,
    "eventTime": 1792656120000
  }
}
```

在同一持续订阅中，旧周期的最终 `isClosed=true` 更新先于下一周期的初始状态发送。如果连接在周期切换时断开，不保证客户端收到最终消息，需要通过 HTTP 补查。

同一根 K 线的推送是完整累计状态：不能把本次 `volume` 加到上一次 `volume` 上，不能把 `tradeCount` 当作本次新增成交笔数。合并规则见第 3.5 节。

### 2.9 应用层心跳

客户端每 20 秒发送一次应用层 ping，整个连接共用一个心跳调度器，无须按频道分别发送。

客户端：

```json
{
  "op": "ping",
  "requestId": "ping-0001",
  "ts": 1792656080000
}
```

服务端：

```json
{
  "op": "pong",
  "requestId": "ping-0001",
  "ts": 1792656080010,
  "echoTs": 1792656080000
}
```

字段与处理规则：

- ping 的 `ts` 是客户端发送时的墙上时钟，`echoTs` 原样回显；pong 的 `ts` 是服务端发送时间。
- ping 的 `ts` 必须为非负毫秒时间戳。
- 客户端以 `requestId` 匹配 ping/pong，不因收到其他行情消息就把未响应的 ping 标记成功。
- 发送 ping 后 5 秒内没有匹配的 pong，客户端关闭旧连接并执行重连。
- 服务端连续 60 秒没有收到该连接的任何有效应用层控制消息时关闭连接，关闭码为 `1001`。
- WebSocket 协议层 ping/pong 可以由客户端库处理，但不能替代本指南约定的 JSON 应用层心跳，也不刷新上述应用层空闲计时。
- 衡量往返耗时使用 `System.nanoTime()` 记录本地发送和接收差值，避免系统时间校准影响超时判断。

### 2.10 断线重连与订阅恢复

客户端应将连接状态至少区分为：连接中、已连接、订阅中、数据就绪、重连等待、主动停止。连接握手成功不代表四类数据都已就绪。

标准重连流程：

1. 原子地将旧连接标记失效，停止旧连接的心跳和重连调度。
2. 将依赖该连接的实时数据标记为过期；保留旧值用于展示时，需要同时提供过期标记。
3. 按退避间隔发起一次新连接，避免同一时刻多个线程重复重连。
4. 新连接成功后，使用新的请求 ID 恢复本地保存的“期望订阅集合”。超过 20 个订阅时分批发送，同时遵守控制消息频率。
5. 等待各批订阅确认，再按频道执行初始化或 HTTP 补查。
6. 各频道达到恢复条件后，分别标记数据可用。

可采用 `1、2、4、8、16、30` 秒封顶的指数退避，并在每次间隔上增加 0～500 ms 随机抖动。连接持续稳定 60 秒后再重置退避计数，避免频繁短连一直按 1 秒重试。

| 频道 | 重连后的恢复条件 |
|---|---|
| 最新价格 | 收到新订阅的最新状态快照 |
| 盘口深度 | 收到新快照并开始按序应用增量；不沿用旧增量链 |
| 逐笔成交 | 新实时流建立，并完成需要的 HTTP 近期成交补查；无法覆盖的缺口单独记录 |
| 实时 K 线 | 新实时流建立，并按最后已保存周期通过 HTTP 补齐断线期间的 K 线 |

每次建立连接都递增客户端本地 `connectionGeneration`。回调和待处理任务携带该代次；旧连接关闭后到达的消息、关闭回调和异步计算结果不得覆盖新连接状态。

### 2.11 WebSocket 错误与关闭码

| 错误码 | 含义 | 处理方式 |
|---|---|---|
| `INVALID_JSON` | 文本不是有效 JSON 对象 | 修正序列化逻辑 |
| `INVALID_OPERATION` | 不支持的 `op` | 仅使用 `subscribe`、`unsubscribe`、`ping` |
| `INVALID_ARGUMENT` | 缺少字段、未知字段、类型或范围错误 | 根据 `details` 修正 |
| `SYMBOL_NOT_FOUND` | 交易对不存在 | 修改交易对 |
| `INVALID_INTERVAL` | K 线周期不支持 | 使用规定周期 |
| `DEPTH_SUBSCRIPTION_CONFLICT` | 同连接同交易对已订阅另一盘口深度 | 先取消原深度，再订阅新深度 |
| `SUBSCRIPTION_LIMIT_EXCEEDED` | 有效订阅将超过 50 个 | 减少或重新分配订阅 |
| `RATE_LIMITED` | 控制消息发送过快 | 合并操作，降低发送频率 |
| `INTERNAL_ERROR` | 服务内部错误 | 有上限重试，持续失败则重连 |

可识别请求时，错误响应的 `op` 和 `requestId` 对应原请求。JSON 无法解析或缺少可用请求 ID 时，服务端使用 `op="error"`、`requestId=null`，其他错误结构与第 2.3.3 节一致。无法解析的消息不改变订阅状态。

| 关闭码 | 本协议中的含义 | 客户端行为 |
|---:|---|---|
| 1000 | 正常关闭 | 主动停止时不重连；若服务端关闭且仍有期望订阅，可退避重连 |
| 1001 | 服务关闭、重启或应用层心跳空闲超时 | 检查关闭说明并重连 |
| 1002 | WebSocket 协议错误 | 检查客户端协议使用 |
| 1003 | 发送了不支持的二进制应用消息 | 改为 UTF-8 文本 JSON |
| 1008 | 持续违反消息频率或其他连接策略 | 降低频率、修正行为后再重连 |
| 1009 | 应用消息超过 1 MiB | 缩小请求或排查异常消息 |
| 1011 | 服务内部故障 | 退避重连 |
| 1006 | 客户端观察到的异常断开状态 | 记录网络信息并重连；该值不是对端在线路中发送的关闭码 |

关闭原因文本用于诊断，不应作为唯一的程序判断依据。某些网络断开只触发错误回调而没有正常关闭回调，两个入口都需要汇入同一套幂等的连接失效处理逻辑。

---

## 3. K 线调用与合并规则

### 3.1 支持的周期及 UTC 对齐方式

| `interval` | 含义 | `intervalMs` | 周期边界示例，均为 UTC |
|---|---|---:|---|
| `1m` | 1 分钟 | 60000 | 每分钟的第 0 秒 |
| `5m` | 5 分钟 | 300000 | 每小时 00、05、10……55 分 |
| `15m` | 15 分钟 | 900000 | 每小时 00、15、30、45 分 |
| `30m` | 30 分钟 | 1800000 | 每小时 00、30 分 |
| `1h` | 1 小时 | 3600000 | 每小时整点 |
| `4h` | 4 小时 | 14400000 | 每日 00、04、08、12、16、20 时 |
| `1d` | 1 天 | 86400000 | 每日 00:00:00 |

周期值区分大小写，`1M`、`60m`、`24h` 均不是合法别名。本文不定义周线、月线或自定义周期。

对于毫秒时间戳 `t` 和周期长度 `P`：

```text
openTime  = floor(t / P) × P
closeTime = openTime + P
成交归属条件：openTime ≤ tradeTime < closeTime
```

一笔恰好发生在 UTC `08:01:00.000` 的成交属于 `08:01` 开始的 1 分钟 K 线，不属于 `08:00` 那根 K 线。

所有日线按 UTC 自然日划分，不随客户端时区或夏令时改变。Java 服务部署在 `Asia/Shanghai` 时，UTC 日线边界显示为北京时间 08:00；显示转换不改变 K 线唯一键和请求时间戳。

### 3.2 时间筛选与分页方向

筛选对象是 **K 线的 `openTime`**，不是该 K 线是否与请求区间发生重叠。服务端不会把请求时间自动取整到周期边界。

当同时传入两个边界时：

```text
startTime ≤ openTime < endTime
```

例如，查询 1 分钟 K 线，范围为 UTC `[08:00:10, 08:01:10)`：

- `08:00:00` 开盘的 K 线不返回，因为其开盘时间早于下界。
- `08:01:00` 开盘的 K 线可以返回，即使其收盘边界在上界之后。
- 如需所有完整位于某时间范围内的 K 线，应自行对齐边界，或进一步判断 `closeTime`。

#### 3.2.1 四种参数组合

| `startTime` | `endTime` | 选取方式 | `pageDirection` | 翻页方式 |
|---|---|---|---|---|
| 不传 | 不传 | 从当前已形成的数据中取最近 `limit` 根 | `backward` | 下一页只传 `endTime=nextEndTime` |
| 不传 | 传入 | 从 `openTime<endTime` 的数据中取最近 `limit` 根 | `backward` | 保持不传 `startTime`，更新 `endTime` |
| 传入 | 不传 | 从 `openTime≥startTime` 的当前已有数据中取最早 `limit` 根 | `forward` | 更新 `startTime=nextStartTime` |
| 传入 | 传入 | 在指定区间内取最早 `limit` 根 | `forward` | 保持原 `endTime`，更新 `startTime` |

无论按哪个方向翻页，单页 `items` 永远按 `openTime` 升序返回。`backward` 表示页与页向更早历史推进，不表示数组倒序。

#### 3.2.2 游标计算

当 `pageDirection="forward"` 且 `hasMore=true`：

```text
nextStartTime = 本页最后一根 K 线的 closeTime
nextEndTime   = null
```

当 `pageDirection="backward"` 且 `hasMore=true`：

```text
nextStartTime = null
nextEndTime   = 本页第一根 K 线的 openTime
```

当 `hasMore=false` 时，两种游标都为 `null`。返回条数正好等于 `limit` 并不能证明还有下一页，必须使用 `hasMore`。

`hasMore` 只描述本次快照中已经存在的数据，不为未来尚未形成的 K 线保留下一页。需要持续更新当前行情时使用 WebSocket，不要在 `hasMore=false` 后无限循环请求同一页。

#### 3.2.3 正向翻页示例

假设 UTC `08:05:00` 之后查询区间 `[08:00:00, 08:05:00)` 的五根已收盘 1 分钟 K 线，`limit=2`：

| 页 | 请求 `startTime` | 保持不变的 `endTime` | 本页开盘时间 | `hasMore` | `nextStartTime` |
|---|---:|---:|---|---|---:|
| 1 | 1792656000000 | 1792656300000 | 08:00、08:01 | true | 1792656120000 |
| 2 | 1792656120000 | 1792656300000 | 08:02、08:03 | true | 1792656240000 |
| 3 | 1792656240000 | 1792656300000 | 08:04 | false | null |

不能将下一页下界设置为上一页最后一根的 `openTime`，否则会重复取到该根 K 线。使用服务端返回的 `nextStartTime` 即可。

#### 3.2.4 反向翻页示例

先不传时间边界查询最近两根，假设当前已形成的最新两根为 `08:00`、`08:01`，且存在更早数据：

```text
第 1 页：interval=1m&limit=2
返回：08:00、08:01
nextEndTime=1792656000000

第 2 页：interval=1m&endTime=1792656000000&limit=2
返回：07:58、07:59
```

向前端追加更早历史时，将第二页插入已有列表的前部，并按 K 线唯一键去重。不要因反向翻页而将单页数组倒置两次。

### 3.3 查询边界与空结果

| 场景 | 处理规则 |
|---|---|
| `startTime=endTime` 或 `startTime>endTime` | HTTP `400`，`INVALID_TIME_RANGE` |
| `startTime` 早于最早可用 K 线 | 正常查询，返回范围内实际存在的数据，不补上市前历史 |
| `endTime` 早于或等于最早可用 K 线 | 成功，`items=[]` |
| `startTime` 在未来，未传 `endTime` | 成功，`items=[]` |
| `endTime` 在未来 | 只返回 `asOf` 时已经形成的 K 线，不生成未来数据 |
| 指定范围内无记录 | `count=0`、`hasMore=false`、两个游标均为 `null` |
| 交易对存在但尚无成交历史 | `availableFrom=null`，`items=[]` |
| 交易对不存在 | HTTP `404`，`SYMBOL_NOT_FOUND` |
| `limit=0`、负数或大于 1000 | HTTP `400`，`INVALID_ARGUMENT`，不自动裁剪 |
| `interval=2m` 等不支持值 | HTTP `400`，`INVALID_INTERVAL` |

本设计不额外限制单次请求的时间跨度，响应仍受最多 1000 根限制。长时间范围使用游标分页，不通过增大 `limit` 绕过上限。

### 3.4 无成交周期、累计值与收盘状态

#### 3.4.1 无成交时如何生成 K 线

交易对已有历史成交后，如果某个完整周期内没有成交，仍生成连续 K 线：

```text
open       = 上一周期 close
high       = 上一周期 close
low        = 上一周期 close
close      = 上一周期 close
volume     = 0
quoteVolume= 0
tradeCount = 0
```

JSON 中价格、数量和金额仍按字符串返回，零成交量可表示为 `"0.00000000"`；客户端按数值判断，不要求零值具有特定小数位数。

新周期开启但尚未成交时，当前 K 线也使用以上延续价格作为暂存状态，`isClosed=false`。如果本周期随后出现第一笔真实成交，则以该笔价格重新确定本周期的 `open/high/low/close`，不把延续价格计入真实成交最高价或最低价。

交易对第一笔历史成交之前不生成 K 线。首笔成交形成的首根 K 线，其 `openTime` 按所属周期向下对齐；不补齐这个周期之前的空白历史。

因此，有成交历史后的周期缺失应当被视为需要检查或补查的缺口，不能默认解释为“当时没有成交”。

#### 3.4.2 成交量和成交额

以 `BTC-USDT` 为例：

- `volume` 的单位为 BTC。
- `quoteVolume` 的单位为 USDT。
- `tradeCount` 为撮合成交笔数。
- `quoteVolume` 为本周期每笔 `price × quantity` 的求和，不等于 `close × volume`。
- `volume>0` 时，可由 `quoteVolume / volume` 计算本周期成交量加权均价；除法精度和舍入方式由客户端明确配置。
- `volume=0` 时不能做该除法，应返回业务空值或使用明确的展示规则。

一根有成交 K 线应满足：

```text
low ≤ open ≤ high
low ≤ close ≤ high
volume > 0
quoteVolume > 0
tradeCount > 0
```

#### 3.4.3 收盘状态与修订号

`isClosed=false` 表示仍可能更新的当前状态；`isClosed=true` 表示服务端已完成该周期收盘。本虚构 v2.1 协议将已收盘 K 线定义为不可再修改，不包含事后历史行情修订机制。

`revision` 的作用域仅为一根 K 线，即 `(symbol, interval, openTime)`：

1. 该根 K 线首次建立时为 `1`。
2. 价格、累计数量、累计金额、成交笔数或收盘状态发生变化时形成更高修订号；内部可批量生成版本，不要求修订号等于成交笔数。
3. HTTP 重复读取和 WebSocket 重发同一状态不会增加修订号。
4. 服务端可以合并中间修订，因此客户端看到 `31` 直接跳到 `97` 是合法的。
5. 不同 K 线的修订号互不比较；新周期可以重新从 `1` 开始。
6. 同一根 K 线相同修订号必须对应相同业务内容，HTTP 和 WebSocket 也遵循这一点。
7. `eventTime` 表示该修订形成时间，不是接收时刻，也不是每次查询时间。

客户端不能仅因本机时间超过 `closeTime` 就把 K 线标记为已收盘。网络延迟、服务端处理和本机时钟偏差都会影响观察顺序，应以服务端字段为准，并在需要时 HTTP 补查。

### 3.5 历史数据与实时数据如何合并

#### 3.5.1 唯一键与覆盖规则

唯一键：

```text
(symbol, interval, openTime)
```

对于同一唯一键：

| 已有状态 | 新到状态 | 处理 |
|---|---|---|
| 没有本地记录 | 任意合法状态 | 新增 |
| 本地未收盘 | 更高 `revision` | 用新状态整体替换 |
| 本地未收盘 | 更低 `revision` | 丢弃过期状态 |
| 任意状态 | 相同 `revision` 且业务值相同 | 忽略重复 |
| 任意状态 | 相同 `revision` 但业务值不同 | 记录协议异常，HTTP 重查，不任意覆盖 |
| 本地已收盘 | 更低修订的未收盘状态 | 丢弃晚到的旧数据 |
| 本地已收盘 | 更高修订或相同修订内容冲突 | 违反已收盘不可修改约定，记录异常并重查 |

比较业务内容时按十进制数值比较价格和数量，不把 `"1.0"` 与 `"1.00"` 视为行情冲突。`symbol`、`interval`、`openTime`、`closeTime`、`isClosed`、成交笔数和修订号则必须准确匹配。

WebSocket 消息中的完整累计值应该整体覆盖，不对 `volume`、`quoteVolume` 或 `tradeCount` 再做累加。

#### 3.5.2 首次加载走势图

推荐流程：

1. 先建立 WebSocket K 线订阅，并缓存订阅确认之后收到的当前快照和更新。
2. 使用 HTTP 获取需要展示的历史 K 线，例如最近 500 根。
3. 将 HTTP 结果按唯一键写入本地缓存或数据库。
4. 按上述修订号规则合并实时缓冲。即使 HTTP 响应较晚到达，也不能覆盖已经收到的更新版本。
5. 将初始化完成前后进入的实时消息串行交给同一合并逻辑，避免“清空缓冲”和“收到新消息”之间产生窗口。
6. 完成后持续处理 WebSocket，使用 HTTP 按需加载更早历史。

缓冲需要设置容量或时长上限。若初始化过慢导致缓冲溢出，应重新执行初始化，不能丢掉部分更新后仍声称数据完整。

#### 3.5.3 断线后的 K 线补齐

恢复订阅并缓存新消息后，从本地最后保存的那根 K 线的 `openTime` 开始进行 HTTP 正向查询，包含该根 K 线本身：

```text
symbol=BTC-USDT
interval=1m
startTime=<本地最后保存的 openTime>
limit=1000
```

这里有意保留一个重叠周期，以便将断线前保存的未收盘版本更新成最终收盘版本。若断线跨越超过 1000 个周期，应按 `nextStartTime` 分页补齐，再合并实时缓冲。

批量补查时可固定一个 `endTime`，让本次追赶范围有限，之后由实时流接管；本协议不提供跨多个 HTTP 请求固定同一数据库快照的令牌。已收盘数据不可变，当前周期通过修订号消除不同请求时刻造成的冲突。

需要对连续性做校验时，在首根可用 K 线之后，期望相邻记录满足：

```text
next.openTime = current.closeTime
```

若不满足，按缺失范围补查。没有成交的周期也应存在零成交量 K 线，因此不能跳过检查。

#### 3.5.4 切换周期或交易对

切换 `BTC-USDT/1m` 到 `BTC-USDT/1h` 时，两者属于不同的数据集合：

- HTTP 历史请求和 WebSocket 订阅都要使用新周期。
- 前端或缓存切换时保留独立的任务代次，旧请求的晚到响应不能覆盖新周期画面。
- 如果不再需要旧周期，取消旧订阅并等待确认，避免长期占用订阅名额。
- 不把不同周期的修订号、成交量或唯一键混合。

---

## 4. Java 对接注意事项与局部示例

以下仅提供 HTTP 调用、JSON 处理、精度处理和订阅发送的局部片段，不提供完整客户端工程。HTTP 片段使用 Java 自带的 `java.net.http` API；JSON 片段以项目已采用 Jackson 的情况为例，不在本文展开依赖配置。

### 4.1 HTTP 查询片段

下面展示 K 线查询和响应读取。片段由已有业务方法调用，调用方负责处理 `IOException`、`InterruptedException` 和业务错误，并按第 1.8 节配置有限重试。

```java
import java.net.URI;
import java.net.http.HttpClient;
import java.net.http.HttpRequest;
import java.net.http.HttpResponse;
import java.time.Duration;
import java.util.UUID;
import com.fasterxml.jackson.databind.JsonNode;
import com.fasterxml.jackson.databind.ObjectMapper;

// HttpClient 和 ObjectMapper 应作为服务级实例复用。
HttpClient client = HttpClient.newBuilder()
        .connectTimeout(Duration.ofSeconds(5))
        .build();
ObjectMapper mapper = new ObjectMapper();

// 使用接入方指定的 HTTP 服务地址。
URI uri = URI.create(
        "https://api.duraqex.com/v2.1/market/klines"
        + "?symbol=BTC-USDT&interval=1m&limit=500");

HttpRequest request = HttpRequest.newBuilder(uri)
        .timeout(Duration.ofSeconds(10))
        .header("Accept", "application/json")
        .header("X-Request-ID", UUID.randomUUID().toString())
        .GET()
        .build();

HttpResponse<String> response = client.send(
        request, HttpResponse.BodyHandlers.ofString());

if (response.statusCode() != 200) {
    // 在业务错误处理层读取 Retry-After、请求 ID 和错误正文。
    // 正文可能是网关返回的非 JSON 内容，不能直接当成功对象解析。
    throw new IllegalStateException("HTTP status=" + response.statusCode());
}

JsonNode root = mapper.readTree(response.body());
if (!"0".equals(root.path("code").asText())) {
    throw new IllegalStateException("API code=" + root.path("code").asText());
}

JsonNode data = root.get("data");
if (data == null || !data.isObject()
        || !data.path("items").isArray()) {
    throw new IllegalStateException("Invalid kline response structure");
}

for (JsonNode item : data.get("items")) {
    // 校验各字段类型后，按 (symbol, interval, openTime) 与 revision 合并。
}
```

上述地址中的交易对和周期是固定安全值。若改为动态输入，应先按协议校验，再使用查询参数编码工具构造 URI，不直接拼接任意外部字符串。

同步 HTTP 查询不要放在 WebSocket 回调线程中；补查使用独立执行器，并将结果交回相应数据流的串行处理队列。

### 4.2 价格、数量和金额的精度

```java
import java.math.BigDecimal;

BigDecimal price = new BigDecimal("68250.10");
BigDecimal quantity = new BigDecimal("0.01500000");
BigDecimal quoteQuantity = price.multiply(quantity);

// 精确十进制结果：1023.7515000000
String plainAmount = quoteQuantity.toPlainString();
```

处理原则：

1. 从 JSON 的数字字符串直接构造 `BigDecimal`，不要先调用 `asDouble()` 或 `Double.parseDouble()`。
2. 不使用 `new BigDecimal(0.1)` 之类以二进制浮点数为输入的构造方式。
3. 数值比较使用 `compareTo()`，例如判断数量是否为零使用 `quantity.signum()==0`。
4. `BigDecimal.equals()` 会比较 scale，`1.0` 与 `1.00` 不相等，不适合直接做价格数值相等判断。
5. 计算均价时明确指定除法精度及舍入规则；不要让展示舍入反向影响原始行情缓存。
6. 持久化时使用容量足够的十进制字段或原始字符串，避免数据库隐式裁剪。具体精度根据实际服务实现确定，不依据本文样例猜测全市场上限。

### 4.3 时间戳处理

```java
import java.time.Instant;
import java.time.ZoneId;
import java.time.ZonedDateTime;

long openTime = 1792656000000L;
Instant instant = Instant.ofEpochMilli(openTime);
ZonedDateTime utcTime = instant.atZone(ZoneId.of("UTC"));
ZonedDateTime shanghaiTime = instant.atZone(ZoneId.of("Asia/Shanghai"));

// utcTime：2026-10-22T08:00Z[UTC]
// shanghaiTime：2026-10-22T16:00+08:00[Asia/Shanghai]
```

使用 `long` 存储毫秒时间戳，不使用 `int`。转换使用 `ofEpochMilli`，不能误用 `ofEpochSecond`。请求、缓存键和数据库中保留原始时间戳，展示层再转换时区。

服务端事件时间与本地接收时间可以用于延迟观测，但两台机器存在时钟偏差，简单相减不等于准确的单向网络延迟。超时与重连计时使用本地单调时钟。

### 4.4 发送订阅消息的局部片段

以下假设已有连接成功的 `java.net.http.WebSocket webSocket`，并复用前面的 `ObjectMapper mapper`：

```java
import com.fasterxml.jackson.databind.node.ArrayNode;
import com.fasterxml.jackson.databind.node.ObjectNode;

ObjectNode subscription = mapper.createObjectNode();
subscription.put("op", "subscribe");
subscription.put("requestId", "sub-java-0001");
ArrayNode args = subscription.putArray("args");

args.addObject()
        .put("channel", "ticker")
        .put("symbol", "BTC-USDT");
args.addObject()
        .put("channel", "depth")
        .put("symbol", "BTC-USDT")
        .put("limit", 20);
args.addObject()
        .put("channel", "trades")
        .put("symbol", "BTC-USDT");
args.addObject()
        .put("channel", "kline")
        .put("symbol", "BTC-USDT")
        .put("interval", "1m");

String payload = mapper.writeValueAsString(subscription);
webSocket.sendText(payload, true);
```

`sendText` 返回的异步结果只表示本次发送操作的完成情况，不表示服务端已经订阅成功。必须通过同一 `requestId` 的 `success=true` 确认消息推进订阅状态。

实际服务中应串行发送订阅、取消订阅和心跳，并观察发送操作的完成结果，避免重叠发送或发送异常被忽略。此片段只展示消息构造，不替代确认、超时和重连处理。

### 4.5 WebSocket 文本回调与消息分发

使用 Java `WebSocket.Listener` 时，重点处理以下细节：

1. 在 `onOpen` 中请求接收消息，例如调用 `webSocket.request(1)`；后续按自身处理能力继续申请接收。不要漏掉需求申请导致连接存在却没有业务回调。
2. `onText` 的 `last=false` 表示当前文本消息还未完整，应按片段顺序累积；只有 `last=true` 时才把完整内容交给 JSON 解析器。
3. 分片缓存限制累计消息大小，并在消息完成或连接失效时清理。大小上限按完整 UTF-8 字节数判断，不把 Java 字符数直接当字节数。
4. JSON 解析、校验和业务处理要避免长时间阻塞网络回调线程。进入异步队列前建立清晰的消息所有权，不能继续复用或修改同一 `StringBuilder`。
5. 消费能力不足时使用有界队列和接收需求控制；不能无上限积压行情。
6. 队列满导致盘口增量无法保留时，应将盘口标记失效并重建；不能随意丢掉某一条增量后继续维护盘口。
7. 对逐笔成交，丢消息意味着可能丢失成交记录，应记录缺口并尝试补查；对 K 线，可以在确认同一唯一键且修订递增时保留最新完整状态，但不能丢掉需要持久化的旧周期收盘状态。
8. 同一盘口订阅和同一 K 线数据序列分别保持有序处理。跨交易对可以分区并行，不能让同一盘口的多个更新在不同线程任意重排。

推荐分发判断顺序：

```text
完整 JSON 消息
  → 有 op：处理 subscribe / unsubscribe 确认、pong 或 error
  → 有 channel：按 ticker / depth / trades / kline 分发
  → 两者都没有或结构冲突：记录协议异常
```

类型校验失败时，不要把缺失价格自动当成 `0`，也不要把缺失 `isClosed` 自动当成 `false`。将必需字段缺失与合法的 `null` 区分开，例如“没有历史成交”的 ticker 允许空成交字段，但普通成交记录不允许空价格。

### 4.6 盘口的 Java 数据结构

可按数值排序保存盘口：

```java
import java.math.BigDecimal;
import java.util.Comparator;
import java.util.NavigableMap;
import java.util.TreeMap;

NavigableMap<BigDecimal, BigDecimal> bids =
        new TreeMap<>(Comparator.reverseOrder());
NavigableMap<BigDecimal, BigDecimal> asks = new TreeMap<>();

// 假设当前正在处理已经通过序列校验的一条买盘变化。
BigDecimal changedPrice = new BigDecimal("68250.00");
BigDecimal changedQuantity = new BigDecimal("0.95000000");

if (changedQuantity.signum() < 0) {
    throw new IllegalArgumentException("Negative depth quantity");
} else if (changedQuantity.signum() == 0) {
    bids.remove(changedPrice);
} else {
    bids.put(changedPrice, changedQuantity);
}
```

这个片段只展示单个档位的绝对值更新。完整业务处理仍需要执行第 2.6 节的快照、订阅代次、`bookId` 和序列校验，并且一条消息的所有买卖档位成功应用后才更新本地 `sequence`。

`TreeMap` 通过数值比较区分价格，因此 `68250.0` 和 `68250.00` 会定位到同一档位。使用 `HashMap<BigDecimal, ...>` 时则需要先统一价格键表示，否则可能因 scale 不同生成两个键。

`TreeMap` 本身不提供并发安全保证。可将单个盘口交给一个串行消费者维护，并向其他业务线程发布不可变快照；发布时同时包含 `bookId`、`sequence` 和有效状态，不让读线程看到应用一半的盘口。

### 4.7 K 线存储与幂等更新

无论使用数据库还是内存缓存，都应按以下字段设计幂等逻辑：

| 目的 | 字段或约束 |
|---|---|
| 唯一识别 | `symbol + interval + openTime` |
| 阻止旧版本覆盖 | 比较 `revision` |
| 识别终态 | `isClosed` |
| 周期连续性检查 | `openTime`、`closeTime`、`intervalMs` |
| 审计及排查 | `eventTime`、本地接收时间、来源 HTTP/WS、连接代次 |

更新条件必须与写入原子执行。例如数据库可以通过唯一索引及带修订号条件的更新实现，内存中可以按唯一键串行处理；不能先在一个线程读取修订号，再由其他线程无条件覆盖。

收到收盘消息后，可以触发依赖收盘价的业务处理，但触发动作也必须按 K 线唯一键幂等，避免 HTTP 补查、重新订阅或应用重启后重复触发。

### 4.8 HTTP 与 WebSocket 的职责划分

| 业务需求 | 使用方式 |
|---|---|
| 页面首次显示最新价格 | HTTP 获取一次，或直接等待 ticker 初始快照 |
| 持续刷新最新价 | WebSocket `ticker` |
| 一次查看当前盘口 | HTTP `/market/depth` |
| 持续维护盘口 | WebSocket `depth`，先快照后增量 |
| 显示最近若干笔成交 | HTTP `/market/trades` 初始化，再合并 WebSocket `trades` |
| 记录实时逐笔成交 | WebSocket `trades`，持久化并按 ID 去重 |
| 首次加载走势图 | HTTP `/market/klines` 与已建立的 WebSocket 缓冲合并 |
| 向更早历史翻页 | HTTP K 线反向分页 |
| 固定区间批量下载 K 线 | HTTP K 线正向分页 |
| 实时更新当前蜡烛及收盘状态 | WebSocket `kline` |
| 断线后恢复盘口 | 重新订阅并使用新 WebSocket 快照 |
| 断线后补齐 K 线 | HTTP 从最后保存周期开始补查，再与实时版本合并 |

### 4.9 常见接入问题

| 现象 | 常见原因 | 处理 |
|---|---|---|
| 已连接，但没有行情 | 没有订阅、订阅失败、未申请 Java 接收需求，或市场暂无更新 | 检查订阅确认、初始快照、`request(1)` 与心跳 |
| 最新价没有显示每一笔变化 | ticker 合并了中间状态 | 需要逐笔时订阅 `trades` |
| 金额出现长尾误差 | 先转成了 `double` | 从原始字符串构造 `BigDecimal` |
| 同一个价格出现两档 | 用带不同 scale 的 `BigDecimal` 作为哈希键 | 使用数值排序映射或规范化键 |
| 盘口数量越来越大 | 把绝对数量当差值累加 | 采用替换和零值删除规则 |
| 五档盘口删一档后只剩四档 | 忽略新补入档位、丢增量或边界处理错误 | 校验完整增量与序列，不自行猜测缺失档位 |
| 盘口序列不连续加一 | 服务端合并了多次底层变化 | 检查 `prevSequence=localSequence`，不检查简单加一 |
| 重连后盘口错乱 | 延续旧快照、旧回调覆盖新连接或缺少序列校验 | 清理旧代次状态，以新快照重新建立盘口 |
| K 线成交量不断翻倍 | 对每次完整累计推送做了累加 | 按唯一键和修订号整体替换 |
| K 线列表出现重复时间 | 用每次消息时间创建新记录 | 唯一键使用 `symbol + interval + openTime` |
| 当前 K 线迟迟不收盘 | 漏掉收盘消息或连接已失效 | 检查心跳并 HTTP 重查该周期 |
| 分页重复最后一根 K 线 | 正向翻页仍使用上一页最后的 `openTime` | 使用 `nextStartTime` |
| 历史页顺序错误 | 将 `backward` 误解成单页倒序 | 单页始终升序，按页向前插入 |
| 日线显示从北京时间 08:00 开始 | 日线按 UTC 划分 | 保持 UTC 聚合，仅转换展示时区 |
| HTTP 返回 429 | 共享 IP 下总权重超限 | 合并读取、缓存结果并遵守 `Retry-After` |
| 断线后的成交记录无法补全 | 缺口超出最近 1000 笔窗口 | 显式记录缺口，不能声称完整补齐 |

### 4.10 对接验证场景

下面这些场景用于验证实现是否符合本指南，不代表真实 Duraqex 服务已经通过测试：

| 场景 | 预期结果 |
|---|---|
| 查询最新价后订阅 ticker | 按成交 ID 合并，晚到旧值不覆盖新值 |
| 订阅同一个 ticker 两次 | 第二次成功，订阅总数不增加，不重发初始快照 |
| 单批订阅含一个非法周期 | 整批失败，原订阅不变 |
| 盘口从 20 档切换到 100 档 | 先取消并确认，再新订阅并获取快照 |
| 人为丢失一个盘口增量 | 下一次连续性检查失败，盘口失效并重新订阅 |
| 盘口价位移出前 N 档 | 收到零数量删除，新进入档位获得完整数量 |
| 在 K 线边界断网后恢复 | HTTP 更新重叠周期终态，并补齐缺失周期 |
| 同一 K 线收到旧 HTTP 结果 | 较低修订号被丢弃 |
| 同一收盘 K 线被读取多次 | 不重复累计、不重复触发收盘业务 |
| 完整周期无成交 | 生成延续价格、零成交量、零成交笔数的 K 线 |
| 收到分片 JSON 文本 | 累积到 `last=true` 后仅解析一次 |
| 旧连接消息在新连接之后到达 | 通过连接代次丢弃 |
| 心跳超过 5 秒无响应 | 关闭旧连接，进入退避重连 |
| HTTP 被限流 | 等待规定时间，不并发立即重试 |

---

**文档边界：** 本指南只定义上述行情 HTTP 与 WebSocket 协议。部署地址采用接入方指定的域名；接口可用性、交易对清单和真实限流额度需要由实际服务实现方确认；本文中的示例不能作为真实平台存在或能力已经上线的证明。
