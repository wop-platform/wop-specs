# WOP SDK 配置与客户端规范

> 版本：v0.3-draft（目标态规格）
> 日期：2026-09-15
> 前版：v0.2-draft（Java 绑定专版，2026-09-11；本版扩展为六语言统一规格，Java 细节迁入附录 A）
> 适用仓库：github.com/wop-platform/wop-{lang}-sdk
> 上游规格：[wop-sdk-spec.md](./wop-sdk-spec.md)（v1.0-ratified，附录 H U3 一站式入口承接本规格）
> 决策编号命名空间：K（本文档专属，与 crypto-spec D、sdk-spec Q/G/E 不冲突）
> 公开真源：[wop-platform/wop-specs · docs/specs/wop-sdk-config-spec.md](https://github.com/wop-platform/wop-specs/blob/main/docs/specs/wop-sdk-config-spec.md)

---

## 0. 定位与上游对齐

### 0.1 核心需求

各语言官方 SDK 为商户提供**一站式网关接入**能力，在 [wop-sdk-spec.md](./wop-sdk-spec.md) 协议核心（分步 API）之上叠加配置层：

1. JSON 配置文件声明凭证与网关地址，加载即校验，错误尽早暴露（fail-fast，启动期可见）；
2. `defaultClient().execute(...)`（各语言惯用命名，见 §2）一行完成 签名 → HTTP → 验签解密；
3. 请求级覆盖凭证 / 超时 / 主网关地址；
4. 主网关固定优先 + 备用域名有序切换（Failover）；
5. 平台回调验签 `verifyCallback`（含凭证覆盖重载）。

本规格**不改变**既有协议核心公开契约（`buildRequest` / `verifyResponse` / `verifyCallback` / `RequestDraft` / `Transport`）。配置层是叠加其上的组织方式，不是替代——`execute` 是既有三步（构造、发送、校验）的组合便利路径（sdk-spec 附录 H U3）。

### 0.2 上游对齐（2026-09-11 已落地）

与 [wop-sdk-spec.md](./wop-sdk-spec.md) 的三处对齐事项已经上游修订 PR 裁决并合入（sdk-spec 附录 H U1–U3）：

| # | 上游条目 | 裁决结果 |
|---|---------|---------|
| U1 | §1.1 `wop-sdk-jdkhttp` 描述勘误 | ✅ 描述性勘误落地 |
| U2 | §1.1 适配器表增补 unirest | ✅ 能力扩张 |
| U3 | 分步 API 与一站式 `execute` 共存 | ✅ 本规格承接各语言一站式入口细则 |

### 0.3 标记约定

- **〔现状〕**：该语言已交付行为，本规格收编为规范；
- **〔目标〕**：待实现行为，各语言附录给出动作与阶段（P0/P1/P2）；
- **〔通用〕**：跨语言一致语义，各语言绑定须等价实现。

---

## 1. 范围

### 1.1 能力（〔通用〕）

- JSON 配置文件加载与校验
- 一站式入口：懒加载配置、传输发现、客户端实例缓存复用
- `execute`：单笔请求 签名 → 发送 → 状态拦截 → 验签解密
- 请求级覆盖：凭证、HTTP 超时、主网关地址（`RequestOptions`，各语言命名见 §2）
- 主网关固定优先 + 备用域名有序 Failover
- 平台回调验签 `verifyCallback`（含凭证覆盖重载）

### 1.2 范围外（〔通用〕）

- 证书仓、调用上报
- 密钥文件路径（仅 inline 字符串；整文件外置经环境变量/语言运行时配置指向）
- 各语言 Web 框架 Starter（仅为文档示例，非 SDK 交付物）
- 域名权重路由、熔断降级
- 自动配置热更新（密钥轮换经 `resetDefault()` + 外部编排，K13）
- 分步 `buildRequest` + `verifyResponse` 公开 API（sdk-spec §1.1/§2 明定能力，本规格不收缩）

---

## 2. 概念 API（各语言惯用映射）

```
ConfigLoader                          # 各语言包路径见 §10 / 附录
  ├─ loadDefault() → Config
  ├─ load(location) → Config
  └─ clearCache()

Config（不可变快照）
  ├─ appKey, suite, merchantPrivateKey, platformPublicKey
  ├─ expiredSeconds, serverRoot, backupServerRoots
  └─ httpClient: { connectTimeout, readTimeout, maxRetryCount }

WopClient / Client（协议客户端 + 一站式入口，K17：不另设独立门面类）
  ├─ static defaultClient()           # 惰性：loadDefault → 传输发现 → 构造；缓存复用
  ├─ static fromConfig(config)
  ├─ static resetDefault()
  ├─ buildRequest / verifyResponse / verifyCallback   # 既有分步 API，语义不变
  ├─ execute(method, path, body, level) → VerifyResult
  ├─ execute(..., options) → VerifyResult
  └─ verifyCallback(headers, body, callbackPath, options) → VerifyResult

RequestOptions（请求级覆盖，商户可见）
  ├─ none() / builder()
  └─ 可覆盖：appKey / suite / 双钥 / expiredSeconds / serverRoot / connectTimeout / readTimeout

Transport（HTTP 适配层，sdk-spec §1.1）
  ├─ send(draft) → TransportResponse              # 既有
  └─ send(draft, call) → TransportResponse        # 新增 default/等价扩展；call 携带 serverRoot 与超时
```

`execute` 语义（单笔调用链，全程同一内部 RequestContext）：

```
ctx = RequestContext.resolve(globalConfig, options)    # 内部；含复校验（§6.3）
draft = buildRequest(method, path, body, level)        # 失败同步抛 WopError（sdk-spec §2.2）
response = transport.send(draft, ctx.toTransportCall()) # §7；非 2xx 在此拦截（§7.5）
return verifyResponse(response, draft)                 # 入向不抛 WopError，返回 VerifyResult
```

- 未配置传输时 `execute` 抛 `WopError.configuration`；分步 API 不受影响。
- `verifyCallback` 凭证覆盖重载仅消费 options 中凭证字段，超时与域名字段忽略（K10）。

---

## 3. 配置文件（〔通用〕）

### 3.1 文件命名与部署

| 文件 | 用途 |
|------|------|
| `config/wopSdkConfig.json` | 商户正式配置（含密钥，**外置部署**，勿提交版本库） |
| `config/wopSdkConfigDefault.json` | SDK 仓库附带模板（占位符） |

推荐部署（K6）：正式配置**外置**于应用制品之外（挂载/复制），经 §4.2 发现机制指向。**不建议**将含密钥的正式配置打入应用制品——密钥会进入构建制品与 CI 缓存。

### 3.2 完整示例

```json
{
  "appKey": "app_001",
  "suite": "WOP-RSA3072-SHA256",
  "merchantPrivateKey": "YOUR_MERCHANT_PRIVATE_KEY",
  "platformPublicKey": "YOUR_PLATFORM_PUBLIC_KEY",
  "serverRoot": "https://gw.example.com/gtsp-wop-gateway",
  "backupServerRoots": [
    "https://gw-backup.example.com/gtsp-wop-gateway"
  ],
  "expiredSeconds": 1800,
  "httpClient": {
    "connectTimeout": 10000,
    "readTimeout": 30000,
    "maxRetryCount": 3
  }
}
```

JSON 字段名与各语言配置模型均使用 **camelCase**。

### 3.3 字段定义

#### 身份与协议

| 字段 | 类型 | 必填 | 默认 | 说明 |
|------|------|------|------|------|
| `appKey` | string | 是 | — | 商户 appKey（`x-wop-appkey`） |
| `suite` | string | 是 | — | `WOP-RSA3072-SHA256` / `WOP-RSA4096-SHA256` / `WOP-SM2-SM3` |
| `merchantPrivateKey` | string | 是 | — | 商户私钥，PEM 或 Base64 单行 |
| `platformPublicKey` | string | 是 | — | 平台公钥，PEM 或 Base64 单行 |
| `expiredSeconds` | long | 否 | `1800` | 出向签名有效窗口（秒，须为正整数） |

#### 网关

| 字段 | 类型 | 必填 | 默认 | 说明 |
|------|------|------|------|------|
| `serverRoot` | string | 是 | — | 主网关根地址，须含 context-path；末尾 `/` 加载时 trim |
| `backupServerRoots` | string[] | 否 | `[]` | 备用网关根地址，**有序，仅排在主网关之后**；仅全局，不可请求级覆盖 |

> 字段名 `backupServerRoots`（K7）：语义为「主网关失败后依序尝试的备用」。

#### HTTP 客户端（`httpClient` 对象）

| 字段 | 类型 | 必填 | 默认 | 说明 |
|------|------|------|------|------|
| `connectTimeout` | int (ms) | 否 | `10000` | TCP 连接超时 |
| `readTimeout` | int (ms) | 否 | `30000` | 读响应超时 |
| `maxRetryCount` | int | 否 | `3` | 跨域名重试上限；仅全局 |

### 3.4 加载校验

加载完成后 fail-fast；错误消息中的字段名与 JSON 一致（camelCase），异常统一 `WopError.configuration`（sdk-spec §2.2）：

| 校验项 | 示例消息 |
|--------|----------|
| JSON 语法错误 | `配置文件 JSON 解析失败: ...` |
| 重复键 / 类型不符 / 数字越界 | `配置字段 expiredSeconds 类型非法: ...` |
| 缺少必填字段 | `配置文件缺少必填项: appKey` |
| `serverRoot` 非法 URL | `serverRoot 不是合法 URL: ...` |
| `suite` 无法识别 | `不支持的算法套件: ...` |
| 密钥格式/长度非法 | `密钥解析失败: ...` |
| `expiredSeconds` ≤ 0 | `expiredSeconds 须为正整数` |

---

## 4. 配置加载（〔通用〕）

### 4.1 ConfigLoader 契约

配置加载入口，线程安全；各语言绑定见 §10 与附录。

| 方法 | 语义 |
|------|------|
| `loadDefault()` | 按 §4.2 自动发现并加载；同一位置缓存解析结果 |
| `load(location)` | 显式位置；`classpath:` 前缀（或各语言等价机制）与文件系统路径须无歧义区分（K14 附则） |
| `load(path)` | 显式文件系统路径 |
| `clearCache()` | 清除加载缓存（测试 / 配置轮换编排） |

- **缓存**：以规范化后的资源位置为 key，同一位置多次加载返回同一 `Config` 实例（不可变）。**无自动失效**（K13）。
- 显式 `load(location)` 带 `classpath:`（或等价前缀）走 classpath，其余一律文件系统路径；禁止「先文件后 classpath」双猜。

### 4.2 自动发现顺序（`loadDefault`，K6）

命中第一个**存在且可读**的来源即加载：

| 优先级 | 来源 |
|--------|------|
| 1 | 语言运行时**显式配置路径**（见 §10 表「配置路径覆盖」列） |
| 2 | 环境变量 `WOP_SDK_CONFIG` |
| 3 | 文件 `{cwd}/config/wopSdkConfig.json` |
| 4 | 文件 `{cwd}/wopSdkConfig.json` |
| 5 | 文件 `{userHome}/.wop/wopSdkConfig.json` |
| 6 | 打包资源 `config/wopSdkConfig.json`（兜底） |

> classpath / 打包资源从早期高位降为**兜底**（K6）：外部配置必须能覆盖打包默认，否则传递依赖制品携带同名资源即静默劫持运行时配置。

- `{cwd}` / `{userHome}`：各语言按惯用 API 解析（Java `user.dir` / `user.home`；Go `os.Getwd()` / `UserHomeDir()`；等）。
- 优先级 1 **已设置但指向不可读** → 立即 `WopError.configuration`（显式指定不容忍），**不**继续后续候选。
- 全部来源未命中 → `WopError.configuration`，消息列出各候选实际展开路径。

### 4.3 解析管道

```
定位资源
  → UTF-8 读入（容忍并剥离 BOM；空文件报错）
  → JSON 解析（未知字段忽略；§4.4 约束）
  → 字段归一化（trim、serverRoot 去尾 /、缺省填默认值）
  → §3.4 语义校验
  → 返回不可变 Config
```

### 4.4 JSON 解析约束（K8，〔通用〕）

- 支持类型：对象、字符串、数字（long/int 范围）、布尔、null、字符串数组、**一层**嵌套对象（`httpClient`）；更深嵌套 → configuration 错误；
- 字符串完整 RFC 8259 转义集（含 `\uXXXX`），字符串内容中的 `}` `]` `,` 不参与边界判定（sdk-spec 附录 D3 纪律）；
- 未知字段忽略；重复键、类型不符、数字越界 → configuration 错误（消息含字段名）；
- **运行时依赖纪律**：配置 schema 为 ~10 个扁平字段，各语言须使用**标准库或内置极简解析器**，禁止为配置层引入重量级 JSON 绑定库（各语言具体约束见附录）。

---

## 5. 一站式入口（〔通用〕，K17）

不引入独立门面类：协议客户端类（`WopClient` 或各语言等价命名）即一站式入口。

| 静态方法 | 语义 |
|----------|------|
| `defaultClient()` | 惰性：`loadDefault` → 传输发现 → 构造；缓存复用同一实例 |
| `fromConfig(config)` | 显式配置构造（不进默认实例缓存） |
| `resetDefault()` | 丢弃默认实例缓存；配合 `clearCache()` 做轮换编排 |

- **并发安全（K15）**：`defaultClient()` 惰性初始化须同步，并发首调仅创建一个实例。
- 密钥轮换：无自动热更新——`clearCache()` + `resetDefault()` + 外部编排，或进程重启（K13）。

---

## 6. 请求级覆盖（〔通用〕）

### 6.1 RequestOptions 可覆盖范围

| 配置项 | 全局 JSON | RequestOptions |
|--------|-----------|----------------|
| `appKey` / `suite` / 双钥 / `expiredSeconds` | ✅ | ✅ |
| `serverRoot` | ✅ | ✅（**整组替换候选，见 K3**） |
| `httpClient.connectTimeout` / `readTimeout` | ✅ | ✅（各适配器能力边界见语言附录） |
| `backupServerRoots` | ✅ | ❌ |
| `httpClient.maxRetryCount` | ✅ | ❌ |

多 appKey：出向经 `RequestOptions` 临时指定另一套凭证；入向回调经 `verifyCallback` 凭证覆盖重载（K10）。

### 6.2 合并规则

| 项 | 规则 |
|----|------|
| 字符串字段（含凭证、serverRoot） | options 已设置且非空 → options，否则 global |
| 超时 | options 值 `> 0` → options，否则 global |
| `expiredSeconds` | options 值 `> 0` → options，否则 global |
| `backupServerRoots` / `maxRetryCount` | 恒 global |
| **Failover 候选** | serverRoot **被覆盖** → 候选 = `[options.serverRoot]`，本笔**关闭 Failover**（K3）；未覆盖 → `[serverRoot] ++ backupServerRoots`（去重保序） |

### 6.3 resolve 复校验（fail-fast）

合并后必须重新执行 §3.4 等价校验——suite 解析、双钥解析与套件族交叉校验、serverRoot URL 合法性。失败同步抛 `WopError.configuration`。

### 6.4 性能

- `RequestOptions.none()`：复用预解析的默认 RequestContext；
- 凭证覆盖：按 `(appKey, suite, keyMaterial)` 缓存密钥解析结果。

---

## 7. HTTP 传输与 Failover（〔通用〕语义 + 语言附录细节）

### 7.1 TransportCall

每笔 `execute` 调用向传输层传递的不可变参数（public 或 package 内，各语言自定；null / -1 = 用适配器默认）：

| 字段 | 说明 |
|------|------|
| `serverRoot` | 目标网关根地址；null = 适配器构造期 baseUrl |
| `connectTimeoutMillis` | -1 = 默认 |
| `readTimeoutMillis` | -1 = 默认 |

- 既有 `send(draft)` 签名/语义不变；新增 `send(draft, call)` 为 default 方法或等价扩展（K2）。
- Failover 候选迭代位于 core 内部，**不修改** public `Transport` 既有单参签名。
- 商户自定义 Transport 仅实现单参 → `execute` 可用（走 default：适配器默认行为、无 Failover、无请求级超时），README 须披露。

### 7.2 传输发现（K12/K18，语义）

| classpath / 依赖面情况 | 行为 |
|------------------------|------|
| 恰一个可用传输实现 | 直接使用 |
| 零个 | `WopError.configuration`（提示引入传输模块） |
| 多个 | 按语言附录选择规则；无法唯一确定时 fail-fast，禁止静默回退（K12） |

各语言发现机制（SPI / 显式 import / 构造注入）见 §10 与附录；**显式指定传输**的 override 名见各语言附录（如 Java `wop.transport`）。

### 7.3 Failover（P2）

```
本笔候选 = serverRoot 被覆盖 ? [options.serverRoot]
                               : [serverRoot] ++ backupServerRoots（去重、保序）
```

| 规则 | 说明 |
|------|------|
| 可重试（仅 pre-send） | 连接阶段失败：DNS 不可达、连接拒绝、路由不可达、**连接**超时（K4：须能判别连接 vs 读超时，无法判别一律不重试） |
| 不可重试 | 读超时（body 已发出，重复提交风险）、任何 HTTP 状态码、协议类 `WopError`、其他 SDK 异常 |
| 上限 | `min(maxRetryCount, 候选数 - 1)` |

全部候选失败 → SDK 异常（`全部网关地址不可用（已尝试 N 个）`，cause = 最后一次异常）。

### 7.4 方法与重定向（〔通用〕原则）

- 3xx 一律按 §7.5 非 2xx 处理；适配器**不得跟随重定向**（改写已签名语义）。
- 扩展 HTTP 方法（如 PATCH）是否支持取决于各语言默认适配器，不支持时 fail-fast；详见语言附录矩阵。
- `Content-Type` 非签名头，缺省 `application/octet-stream`。

### 7.5 响应状态语义（K5）

`execute` 链路中 HTTP 状态码非 2xx → **不进入验签**，抛网关响应异常（各语言命名自定，须含 `statusCode` 与 `body` 访问器）：

- message 格式：`WOP 网关返回 HTTP <code>（响应体 N 字节）`——不内嵌 body 全文（防日志膨胀）。

### 7.6 响应体上限

流式读取上限 **11MB**（`11 << 20` 字节，sdk-spec 附录 D4）：读取过程中逐块计数、超限即断流。任何传输实现必须同构。

### 7.7 path 语法（K14）

`execute` / `buildRequest` 的 `path` 必须是 `/` 开头的**纯 API 路径**（如 `/gateway/order/create`），相对 `serverRoot` / `TransportCall.serverRoot` 解析；**不含 scheme 与域名**，亦不得携带 query / fragment。以下任一情形即 `WopError.configuration` fail-fast：

- 含 `?` 或 `#`（sdk-spec 附录 G1：canonicalQueryString 恒空）；
- 含 scheme / 域名（绝对 URL，或任何非 `/` 开头形式）。

---

## 8. 错误契约（〔通用〕）

| 场景 | 结果 | 类别（sdk-spec §2.2） |
|------|------|----------------------|
| 配置加载 / 校验 / resolve 复校验失败 | `WopError` | configuration（明确） |
| 出向构造失败 | `WopError` | 同 `buildRequest` 既有契约 |
| 非 2xx 响应 | 网关响应异常 | 系统类（明确，含状态码） |
| 全部网关不可达 / 单笔传输失败且不再重试 | SDK 异常 | 系统类（明确） |
| 验签 / 解密失败 | `VerifyResult.ok() == false` | I7 模糊 |

---

## 9. 安全（〔通用〕）

| 要求 | 说明 |
|------|------|
| 密钥不落日志 | `Config.toString` 打码；异常、debug 日志同纪律（K16） |
| 文件部署 | 正式配置外置（K6），生产建议 restrictive 文件权限；不得提交版本库 |
| 回调时间窗 | 签名覆盖 `x-wop-timestamp`，**新鲜度校验不在 SDK 职责**（K10），由商户业务层判定 |

---

## 10. 各语言绑定概览

| 语言 | 仓库 | 客户端 / ConfigLoader | 默认传输适配器（sdk-spec §1.1） | 配置路径覆盖（优先级 1） | 传输显式指定 | 实现状态 |
|------|------|----------------------|--------------------------------|-------------------------|--------------|----------|
| Java | wop-java-sdk | `WopClient` / `WopSdkConfigLoader` | jdkhttp / okhttp / unirest（SPI） | JVM `-Dwop.sdk.config.file` | `wop.transport` | 分步 API 〔现状〕；配置层 〔目标〕，见附录 A |
| Go | wop-go-sdk | `Client` / `ConfigLoader`（惯用命名） | 默认 `http.Client` + `RoundTripper` 桥接 | 环境变量 `WOP_SDK_CONFIG_FILE` | `WOP_TRANSPORT` 或构造注入 | 〔目标〕，见附录 B |
| TypeScript | wop-typescript-sdk | `WopClient` / `loadConfig` | fetch 原生 + axios peer | 环境变量 `WOP_SDK_CONFIG_FILE` | 模块 import 或 env | 〔目标〕，见附录 C |
| Python | wop-python-sdk | `WopClient` / `ConfigLoader` | stdlib urllib + httpx/requests peer | 环境变量 `WOP_SDK_CONFIG_FILE` | 构造参数或 env | 〔目标〕，见附录 D |
| PHP | wop-php-sdk | `WopClient` / `ConfigLoader` | curl + Guzzle peer | `getenv('WOP_SDK_CONFIG_FILE')` | 构造参数 | 〔目标〕，见附录 E |
| .NET | wop-dotnet-sdk | `WopClient` / `WopConfigLoader` | `HttpClient` + DelegatingHandler | 环境变量 `WOP_SDK_CONFIG_FILE` | 构造注入或配置节 | 〔目标〕，见附录 F |

> 环境变量 `WOP_SDK_CONFIG`（优先级 2）六语言统一：指向配置文件路径。优先级 1 为各语言惯用的**显式覆盖**机制，与 `WOP_SDK_CONFIG` 并存时优先级 1 优先。

---

## 11. 验收标准（每仓通用，配置层增量）

在 [wop-sdk-spec.md](./wop-sdk-spec.md) §5 A1–A7 基础上，配置层合入须额外满足：

| # | 验收项 | 判定 |
|---|--------|------|
| C1 | 配置发现顺序 | §4.2 六来源 + 显式不可读即报错 + 全未命中错误消息 |
| C2 | 加载校验 | §3.4 全表 + 重复键 / BOM / 空文件 |
| C3 | execute 组合链 | 签名 → 发送 → 非 2xx 拦截 → 验签，复用分步 API 不另定义协议 |
| C4 | 请求级覆盖 | §6 合并 / 复校验 / K3 Failover 关闭 |
| C5 | path 语法 | §7.7 纯 API 路径拒绝规则 |
| C6 | 密钥不打日志 | toString / 异常 / debug 打码（K16） |
| C7 | Gherkin（sdk-spec E3） | 配置加载、execute L0/L2、回调验签场景 ≥10；平台响应构造遵守 D5 |

---

## 12. 决策记录（K 系列，〔通用〕）

| # | 决策 | 依据 |
|---|------|------|
| K1 | 文档性质为**目标态规格 + 各语言现状锚点 + 差距台账** | 避免以现在时描述未交付能力 |
| K2 | `Transport` 既有单参签名不动；新增 default 双参 + `TransportCall`；内部 `RequestContext` 永不出现在 public 签名 | 保护商户自定义 Transport 与分步 API |
| K3 | 请求级 serverRoot 覆盖 = 候选集整组替换，本笔关闭 Failover | 防止跨环境凭证漂移 |
| K4 | 重试仅限 pre-send 失败；连接/读超时须可判别，不可判别不重试 | 非幂等 POST 重复提交风险 |
| K5 | 非 2xx 不进验签，抛含 statusCode + body 的网关响应异常 | 错误页不应以验签类 reason 误导排障 |
| K6 | 发现顺序外部优先（显式覆盖 > env > cwd > userHome > 打包资源兜底）；正式配置外置 | 传递依赖制品静默劫持 + 密钥入制品 |
| K7 | 字段名 `backupServerRoots` | 语义为有序备用，非主优先 |
| K8 | 配置解析用标准库或内置极简读取器；禁止重量级 JSON 绑定库进运行时依赖面 | ~10 扁平字段；D3 解析纪律先例 |
| K9 | 传输适配器能力边界须 README/Javadoc 诚实披露 | 如实例级连接超时无请求级 API |
| K10 | `verifyCallback` 凭证覆盖重载；时间窗校验归属商户业务层 | 多 appKey 入向验签 |
| K11 | Builder / 构造器扩展全字段 + 可选 Transport；程序化与 JSON 等价 | 无 serverRoot 则 client 无网关地址 |
| K12 | 传输选择非法/歧义 fail-fast 列出可用项，禁止静默回退 | 掩盖装配错误 |
| K13 | 配置缓存无自动失效；轮换 = clearCache + resetDefault + 外部编排 | 避免「缓存即热更新」误读 |
| K14 | `load(location)` 语义钉死；`path` = `/` 开头纯 API 路径 | G1 canonicalURI；K3 同源候选 |
| K15 | `defaultClient()` 并发首调单实例 | 多 goroutine/线程安全 |
| K16 | 配置与客户端 toString 私钥打码为 P0 | 密钥泄漏防护 |
| K17 | 不引入独立门面类；协议客户端类即一站式入口 | 减少重叠公开面 |
| K18 | 传输发现 P0 须可无环装配默认传输；多实现选择规则各语言附录钉死 | 如 Java ServiceLoader SPI |
| K19 | v0.3 扩展六语言统一规格；Java v0.2 正文迁入附录 A，通用 JSON/语义层上浮正文 | 与 wop-sdk-spec 治理对齐 |

---

## 附录 A：Java 绑定

> 事实基准（2026-09-11）：wop-java-sdk main @ 0.1.0（包根 `com.wanlianyida.wop`，根 pom `maven.compiler.release=8`）

### A.1 现状锚点

| 现状 | 证据 |
|------|------|
| `WopClient` 公开面：`builder()` / `buildRequest` / `verifyResponse` ×2 / `verifyCallback`；无 `execute`，无 Transport 字段 | wop-sdk-core `WopClient.java` |
| `Transport.send(RequestDraft)` 单参公开接口；jdkhttp / okhttp / unirest 三适配器已交付 | `Transport.java` + 三适配器模块 |
| 配置层（`WopSdkConfigLoader`、`WopRequestOptions`、`WopRequestContext`、Failover、SPI）零实现 | git 全历史无门面提交 |
| D4 响应体 11MB 流式上限三适配器均已实现 | 各适配器 `MAX_RESPONSE_BYTES = 11 << 20` |
| jdkhttp：连接超时硬编码 10s，**未设读超时**；拒绝扩展方法（PATCH 等） | `JdkHttpTransport` |
| okhttp：使用 OkHttp 默认（**跟随重定向**）；超时属商户注入实例 | `OkHttpTransport` |
| unirest：连接超时 10s 实例级；关闭 gzip 协商 | `UnirestTransport` |
| `verifyResponse` 丢弃 `statusCode` | `WopClient.verifyInbound` |
| `WopClient.Config.toString()` 明文打印 merchantPrivateKey（K16 缺陷） | `WopClient.java` |
| jackson-databind 为 core **test scope**，主源码集零使用 | core pom |

### A.2 模块与依赖

| 模块 | 职责 | 运行时 |
|------|------|--------|
| `wop-sdk-core` | 配置加载〔目标〕、`WopClient`（+ `execute` 与静态入口）、`WopRequestOptions`〔目标〕、`Transport` + `TransportFactory`（SPI）〔目标 P0〕、Failover〔目标〕 | Java 8+ |
| `wop-sdk-jdkhttp` | `HttpURLConnection` 传输〔现状〕 + SPI 注册〔目标 P0〕 | Java 8+ |
| `wop-sdk-okhttp` | OkHttp 传输〔现状〕 + SPI 注册〔目标 P0〕 | Java 8+ |
| `wop-sdk-unirest` | Kong Unirest 4.x 传输〔现状〕 + SPI 注册〔目标 P0〕 | **运行时 Java 11+** |

商户 `pom` 引入 `wop-sdk-core` + 传输模块之一（通常 `wop-sdk-jdkhttp`）。

**K8（Java）**：配置解析使用 core 内置极简 JSON 读取器（§4.4），不引入 jackson-databind 运行时依赖。

### A.3 传输 SPI（K18）

`java.util.ServiceLoader` 加载 `TransportFactory`（接口在 core，适配器 `META-INF/services` 注册）：

| classpath 情况 | 行为 |
|----------------|------|
| 恰一个 factory | 直接使用 |
| 零个 | `WopError.configuration`（提示引入传输模块） |
| 多个（P2） | 含 jdkhttp → 默认 jdkhttp；不含 → 须 `-Dwop.transport=` 显式指定 |

`wop.transport` 值无匹配 → `WopError.configuration`，列出全部可用 factory 名（K12）。

**配置路径覆盖（优先级 1）**：JVM 系统属性 `wop.sdk.config.file`（`WopSdkConfigLoader.CONFIG_FILE_PROPERTY`）。

### A.4 Java API 签名

```java
package com.wanlianyida.wop.config;

public final class WopSdkConfigLoader {
    public static final String CONFIG_FILE_PROPERTY = "wop.sdk.config.file";
    public static final String CONFIG_FILE_ENV = "WOP_SDK_CONFIG";
    public static WopSdkConfig loadDefault();
    public static WopSdkConfig load(String location);
    public static WopSdkConfig load(java.nio.file.Path path);
    public static void clearCache();
}

public final class WopClient {
    public static WopClient defaultClient();
    public static WopClient fromConfig(WopSdkConfig config);
    public static void resetDefault();
    public VerifyResult execute(String method, String path, byte[] body, SecurityLevel level);
    public VerifyResult execute(String method, String path, byte[] body,
                                SecurityLevel level, WopRequestOptions options);
    public VerifyResult verifyCallback(java.util.Map<String, String> headers, byte[] body,
                                       String callbackPath, WopRequestOptions options);
}
```

Classpath 读取：`Thread.currentThread().getContextClassLoader()` 优先，回退 Loader 类 ClassLoader。

### A.5 适配器矩阵

| 配置项 | JdkHttpTransport | OkHttpTransport | UnirestTransport |
|--------|------------------|-----------------|------------------|
| `connectTimeout` | 每请求 `setConnectTimeout` | 克隆 client `.connectTimeout` | **实例级**（K9，请求级不生效） |
| `readTimeout` | 每请求 `setReadTimeout` | 克隆 client `.readTimeout` | 每请求（K9） |
| PATCH 等扩展方法 | **fail fast 拒绝** | 支持 | 支持 |
| 重定向 | 不跟随〔现状〕 | **须改**为不跟随 | 不跟随〔现状〕 |
| 默认读超时 | 〔现状〕未设 → 须落地 30000 | 10000/30000 | 10000/30000 |

**K9**：Unirest 连接超时为实例级，请求级 `connectTimeout` 覆盖不生效，README/Javadoc 必须披露。

非 2xx 抛 `WopGatewayResponseException`（`statusCode()` + `body()`）。

### A.6 商户接入示例

Maven：

```xml
<dependency>
  <groupId>com.wanlianyida</groupId>
  <artifactId>wop-sdk-core</artifactId>
  <version>${wop.sdk.version}</version>
</dependency>
<dependency>
  <groupId>com.wanlianyida</groupId>
  <artifactId>wop-sdk-jdkhttp</artifactId>
  <version>${wop.sdk.version}</version>
</dependency>
```

```bash
export WOP_SDK_CONFIG=/etc/wop/wopSdkConfig.json
# 或
java -Dwop.sdk.config.file=/etc/wop/wopSdkConfig.json -jar app.jar
```

```java
VerifyResult result = WopClient.defaultClient().execute(
        "POST", "/gateway/order/create", body, SecurityLevel.L0);
```

Spring Boot：`@Bean WopClient wopClient() { return WopClient.defaultClient(); }`

### A.7 实现差距台账

| # | 能力 | 现状 | 动作 | 阶段 |
|---|------|------|------|------|
| 1 | 配置加载 / 校验 / 仓库模板 | 无 | 新增（内置极简解析器） | P0 |
| 2 | `WopClient` 静态一站式入口 | 无 | 新增（K17/K15） | P0 |
| 3 | `execute` ×2 | 无 | 新增 | P0 |
| 4 | `WopClient.Config.toString` 打码 | 明文私钥 | 修复 + 回归 | P0 |
| 5 | `WopRequestOptions` / `WopRequestContext` | 无 | 新增 | P1 |
| 6 | `verifyCallback` 四参重载 | 无 | 新增 | P1 |
| 7 | `Transport.send` 双参 + `TransportCall` | 无 | 增量扩展 | P1 |
| 8 | 三适配器双参实现 | 仅单参 | 实现 | P1 |
| 9 | jdkhttp 读超时默认 30000 | 未设 | 修改 + 回归 | P1 |
| 10 | okhttp `followRedirects(false)` | 跟随 | 修改 | P1 |
| 11 | 非 2xx 拦截 + `WopGatewayResponseException` | statusCode 丢弃 | 新增 | P1 |
| 12 | resolve 复校验 | 无 | 新增 | P1 |
| 13 | `FailoverTransport` | 无 | 新增 | P2 |
| 14 | `TransportFactory` SPI + META-INF/services | 无 | 新增（K18） | P0 |
| 15 | `wop.transport` 多 factory 选择 | 无 | 新增 | P2 |
| 16 | D4 11MB / 方法白名单 / octet-stream | 已实现 | 收编 | — |
| 17 | path 校验（§7.7） | 无 | 新增 | P0 |

阶段：**P0** = 最小可用一站式（SPI、无 Failover、候选仅 serverRoot）；**P1** = 请求级覆盖 + 状态语义 + 适配器补齐；**P2** = Failover 与多传输选择。

### A.8 测试要点（Java）

1. 发现顺序六来源 + SPI 零/多 fail-fast + `wop.transport` 指定/非法值
2. 缓存 / `clearCache()` / `resetDefault()`；`defaultClient()` 并发首调单实例
3. 加载校验全表 + 极简解析器边界（D3）
4. resolve 复校验 + K3 Failover 关闭
5. 非 2xx / 3xx 不进验签；`WopGatewayResponseException` 访问器
6. Failover pre-send 可重试 / 读超时不重试
7. jdkhttp PATCH 拒绝；三适配器均不跟随重定向
8. K16 toString 打码回归
9. path 语法 §7.7
10. Gherkin ≥10 场景（sdk-spec E3）

---

## 附录 B：Go 绑定（〔目标〕）

| 项 | 约定 |
|----|------|
| 包布局 | 主模块 `wop-go-sdk`；传输：`transport/http`（默认 `http.Client`）、可选 `transport/roundtripper` 桥接 |
| ConfigLoader | `config.LoadDefault()` / `config.Load(path)`；`config.ClearCache()` |
| 客户端 | `wop.Client`；`wop.DefaultClient()` / `wop.NewFromConfig(cfg)` / `wop.ResetDefault()` |
| 配置路径覆盖 | 环境变量 `WOP_SDK_CONFIG_FILE`（优先级 1）；`WOP_SDK_CONFIG`（优先级 2） |
| 传输发现 | 默认 `http.DefaultClient` 包装；`WOP_TRANSPORT=http` 显式；商户 `RoundTripper` 注入 |
| JSON 解析 | 标准库 `encoding/json` + 结构体 tag；未知字段 `json:"-"` 或 Decoder `DisallowUnknownFields` 按 K8 取舍——**须忽略未知字段** |
| 并发 | `sync.Once` 保护 `DefaultClient()` |
| 网关响应异常 | `*wop.GatewayResponseError`（`StatusCode` / `Body`） |
| Failover | P2；连接失败可重试判别同 K4（`net.Error` Timeout + `Temporary` 语义） |

---

## 附录 C：TypeScript 绑定（〔目标〕）

| 项 | 约定 |
|----|------|
| 包布局 | `wop-typescript-sdk`；`transport/fetch`（默认）+ `transport/axios`（peer） |
| ConfigLoader | `loadDefault()` / `load(location)` / `clearCache()` |
| 客户端 | `WopClient.defaultClient()` / `fromConfig()` / `resetDefault()` |
| 配置路径覆盖 | `process.env.WOP_SDK_CONFIG_FILE`；`WOP_SDK_CONFIG` |
| 传输发现 | 默认 fetch；多适配器时 env `WOP_TRANSPORT=fetch|axios` 或构造注入 |
| JSON 解析 | `JSON.parse` + 运行时类型校验（zod 等**禁止**进运行时依赖面，手写校验） |
| 并发 | 模块级 Promise 锁或 `AsyncLocalStorage` 等价，首调单实例 |
| 网关响应异常 | `WopGatewayResponseError` |

---

## 附录 D：Python 绑定（〔目标〕）

| 项 | 约定 |
|----|------|
| 包布局 | `wop-python-sdk`；`transport/urllib`（默认）+ `transport/httpx` / `transport/requests`（peer） |
| ConfigLoader | `wop.config.load_default()` / `load()` / `clear_cache()` |
| 客户端 | `WopClient.default_client()` / `from_config()` / `reset_default()` |
| 配置路径覆盖 | `os.environ["WOP_SDK_CONFIG_FILE"]`；`WOP_SDK_CONFIG` |
| 传输发现 | 默认 urllib；`WOP_TRANSPORT=urllib|httpx|requests` 或构造注入 |
| JSON 解析 | 标准库 `json` |
| 并发 | `threading.Lock` 保护 `default_client()` |
| 网关响应异常 | `WopGatewayResponseError` |

---

## 附录 E：PHP 绑定（〔目标〕）

| 项 | 约定 |
|----|------|
| 包布局 | `wop-php-sdk`；curl 适配器 + Guzzle peer |
| ConfigLoader | `WopConfigLoader::loadDefault()` 等 |
| 客户端 | `WopClient::defaultClient()` / `fromConfig()` / `resetDefault()` |
| 配置路径覆盖 | `getenv('WOP_SDK_CONFIG_FILE')`；`WOP_SDK_CONFIG` |
| 传输发现 | 默认 curl；构造注入 Guzzle `Client` |
| JSON 解析 | `json_decode` + 手写校验（禁止 Symfony Serializer 等重量级依赖进 core） |
| 网关响应异常 | `WopGatewayResponseException` |

---

## 附录 F：.NET 绑定（〔目标〕）

| 项 | 约定 |
|----|------|
| 包布局 | `wop-dotnet-sdk`；`HttpClient` + 可选 `DelegatingHandler` |
| ConfigLoader | `WopConfigLoader.LoadDefault()` 等 |
| 客户端 | `WopClient.DefaultClient` / `FromConfig()` / `ResetDefault()` |
| 配置路径覆盖 | 环境变量 `WOP_SDK_CONFIG_FILE`；`WOP_SDK_CONFIG`；可选 `appsettings.json` 节 `WopSdk`（文档示例，非必须） |
| 传输发现 | 默认 `HttpClient`；`WOP_TRANSPORT` 或 DI 注入 |
| JSON 解析 | `System.Text.Json`（`JsonSerializer.Deserialize` + 手写校验） |
| 并发 | `Lazy<WopClient>` 或 `SemaphoreSlim` |
| 网关响应异常 | `WopGatewayResponseException` |
| HttpClient | `AllowAutoRedirect = false`；响应体流式 11MB 限额 |

---
