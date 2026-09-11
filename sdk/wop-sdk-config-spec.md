# WOP SDK 配置与客户端规范（Java 绑定）

> 版本：v0.2-draft（目标态规格）
> 日期：2026-09-11
> 前版：v0.1 草案（未合入，被本版整体取代；修订依据：2026-09-11 基于 wop-java-sdk 实现的第一性原理审查，逐项见 §14 决策记录）
> 同日修订（v0.2 内）：采信三项评审意见——SPI 前移 P0（K18）、path 语法改判纯 API 路径（K14）、门面并入 WopClient（K17）
> 事实基准：wop-java-sdk main @ 0.1.0（包根 `com.wanlianyida.wop`，根 pom `maven.compiler.release=8`）
> 决策编号命名空间：K（本文档专属，与 crypto-spec D、sdk-spec Q/G/E 不冲突）
> 治理：本文件尚未登记 README 目录表——合入 PR 时须按仓库「规格治理」第 4 条同步登记；合入前置依赖见 §0.2

## 0. 定位与上游对齐

### 0.1 核心需求（承自 v0.1，不变）

商户**一站式网关接入**：

1. JSON 配置文件声明凭证与网关地址，加载即校验，错误尽早暴露（fail-fast，启动期可见）；
2. `WopClient.defaultClient().execute(...)` 一行完成 签名 → HTTP → 验签解密；
3. 请求级覆盖凭证 / 超时 / 主网关地址；
4. 主网关固定优先 + 备用域名有序切换；
5. 平台回调验签 `verifyCallback`。

本规格**不改变**既有协议核心公开契约（`buildRequest` / `verifyResponse` / `verifyCallback` / `RequestDraft` / `Transport`）。配置层是叠加其上的组织方式，不是替代——`execute` 是既有三步（构造、发送、校验）的组合便利路径（前置 U3）。

### 0.2 上游对齐前置（2026-09-11 已落地，不再是阻断条件）

本规格与 [wop-sdk-spec.md](./wop-sdk-spec.md)（v1.0-ratified）的三处对齐事项已经上游修订 PR 裁决并合入（[PR #17](https://github.com/wop-platform/wop-specs/pull/17)，合并提交 `4265da3`，sdk-spec 附录 H；正文 §1.1 适配器表与 §2 概念 API 已同步）：

| # | 上游条目 | 原现状 | 裁决结果 |
|---|---------|------|---------|
| U1 | §1.1 `wop-sdk-jdkhttp`（java.net.http，零依赖） | 实现已改建 `HttpURLConnection`（Java 8 floor） | ✅ 描述性勘误落地（附录 H U1） |
| U2 | §1.1 适配器表无 unirest | `wop-sdk-unirest` 已交付 | ✅ 增补入表（附录 H U2，能力扩张） |
| U3 | §1.1「商户自带栈时直接消费 RequestDraft」+ §2 概念 API | v0.1 曾将分步 API 列为范围外 | ✅ 共存声明（附录 H U3 + §2 增补） |

### 0.3 现状锚点（2026-09-11 审查事实，本规格的事实基准）

| 现状 | 证据 |
|------|------|
| `WopClient` 公开面：`builder()` / `buildRequest` / `verifyResponse` ×2 / `verifyCallback`；无 `execute`，无 Transport 字段 | wop-sdk-core `WopClient.java` |
| `Transport.send(RequestDraft)` 单参公开接口；jdkhttp / okhttp / unirest 三适配器已交付 | `Transport.java` + 三适配器模块 |
| v0.1 所述配置层（`WopSdk` 门面、`WopSdkConfigLoader`、`WopRequestOptions`、`WopRequestContext`、Failover、SPI）零实现，且 git 全历史从未存在 | `git log --all --diff-filter=A` 无命中 |
| D4 响应体 11MB 流式上限三适配器均已实现 | 各适配器 `MAX_RESPONSE_BYTES = 11 << 20` |
| jdkhttp：连接超时硬编码 10s，**未设读超时**（默认无限挂起）；拒绝扩展方法（PATCH 等） | `JdkHttpTransport` |
| okhttp：使用 OkHttp 默认（**跟随重定向**，与 jdkhttp 不一致）；超时属商户注入实例 | `OkHttpTransport` |
| unirest：连接超时 10s 实例级；关闭 gzip 协商 | `UnirestTransport` |
| `verifyResponse` 丢弃 `statusCode`，非 2xx 一律进入验签路径 | `WopClient.verifyInbound` 只消费 headers/body/path |
| `WopClient.Config.toString()` 明文打印 merchantPrivateKey（缺陷，修复锚见 K16） | `WopClient.java` |
| jackson-databind 为 core **test scope**（向量/测试使用），主源码集零使用 | core pom + 全仓 grep |

### 0.4 标记约定

- **〔现状〕**：已交付行为，本规格收编为规范（无代码动作或仅有标注的修改）；
- **〔目标〕**：待实现行为，§12 差距台账给出动作与阶段（P0/P1/P2）。

---

## 1. 范围

### 1.1 能力

- JSON 配置文件加载与校验〔目标 P0〕
- 一站式入口 `WopClient.defaultClient()`：懒加载配置、SPI 发现传输、缓存复用（K17/K18）〔目标 P0〕
- `WopClient.execute`：单笔请求 签名 → 发送 → 状态拦截 → 验签解密〔目标 P0〕
- 请求级覆盖：凭证、HTTP 超时、主网关地址（`WopRequestOptions`）〔目标 P1〕
- 主网关固定优先 + 备用域名有序 Failover〔目标 P2〕
- 平台回调验签 `verifyCallback`（含凭证覆盖重载）〔现状 + 目标 P1〕

### 1.2 范围外

- 证书仓、调用上报
- 密钥文件路径（仅 inline 字符串；整文件外置经环境变量/系统属性指向）
- Spring Boot Starter（Spring Bean 装配仅为文档示例）
- 域名权重路由、熔断降级
- 自动配置热更新（密钥轮换经 `WopClient.resetDefault()` + 外部编排，K13）
- ~~分步 `buildRequest` + `verifyResponse` 公开 API~~（v0.1 错误列为范围外——该 API 为已交付公开契约与 sdk-spec §1.1/§2 明定能力，本规格不收缩，见前置 U3）

---

## 2. 模块与依赖

| 模块 | 职责 | 运行时 |
|------|------|--------|
| `wop-sdk-core` | 配置加载〔目标〕、`WopClient`（既有 + `execute` 与静态一站式入口增量）、`WopRequestOptions`〔目标〕、`WopRequestContext`（内部）、`Transport` 接口（既有 + 增量）、`TransportFactory`（SPI 接口）〔目标 P0〕、Failover〔目标〕 | Java 8+ |
| `wop-sdk-jdkhttp` | JDK `HttpURLConnection` 传输〔现状〕 + `TransportFactory` SPI（`META-INF/services` 注册）〔目标 P0〕 | Java 8+ |
| `wop-sdk-okhttp` | OkHttp 传输〔现状〕 + `TransportFactory` SPI（`META-INF/services` 注册）〔目标 P0〕 | Java 8+ |
| `wop-sdk-unirest` | Kong Unirest 4.x 传输〔现状〕 + `TransportFactory` SPI（`META-INF/services` 注册）〔目标 P0〕 | **运行时 Java 11+**（上游 unirest 字节码要求；模块自身 release 8） |

商户 `pom` 引入 `wop-sdk-core` + 传输模块之一（通常 `wop-sdk-jdkhttp`）。

### 2.1 依赖约束（K8）

`wop-sdk-core` 运行时第三方依赖**维持现状**：仅 `bcprov-jdk18on`（sdk-spec §3 白名单）。配置解析使用**内置极简 JSON 读取器**（§4.4），不引入 jackson-databind 运行时依赖；jackson 维持 test scope（黄金向量与测试消费）。

拒绝 jackson 运行时的理由：配置 schema 为 ~10 个扁平字段，databind 的依赖面与跨版本冲突面不成比例；core 已有 D3 字符串感知解析纪律的先例（`EncryptedEnvelope`）。

### 2.2 传输实现选择（SPI，K18）

依赖方向为 适配器 → core：core 无法编译依赖 `JdkHttpTransport`，默认传输的构造唯一无环路径是 ServiceLoader SPI——`TransportFactory` 接口（置于 core）与三适配器的 `META-INF/services` 注册随 **P0** 落地（K18：v0.2 初稿「P0 固定 jdkhttp、SPI 留 P2」不可实现）。

〔目标 P0〕`java.util.ServiceLoader` 加载 `TransportFactory`：

| classpath 情况 | 行为 |
|----------------|------|
| 恰一个 factory | 直接使用 |
| 零个 | `WopError.configuration`（提示引入传输模块，消息列出查找方式） |
| 多个 | `WopError.configuration`（多 factory 选择规则 P2 落地，见下） |

〔目标 P2〕多 factory 选择：

| classpath 情况 | 行为 |
|----------------|------|
| 多个 factory，含 jdkhttp | **默认 jdkhttp** |
| 多个 factory，jdkhttp 不在列 | `WopError.configuration`，要求显式设置 `wop.transport`（ServiceLoader 枚举序不稳定，禁止「取首个」） |
| 任意情况需指定 | JVM 系统属性 `wop.transport=jdkhttp` / `okhttp` / `unirest` |

`wop.transport` 值无匹配 factory → `WopError.configuration`（fail-fast，消息列出全部可用 factory 名），禁止静默回退默认（K12）。

---

## 3. 配置文件

### 3.1 文件命名与部署

| 文件 | 用途 |
|------|------|
| `config/wopSdkConfig.json` | 商户正式配置（含密钥，**外置部署**，勿提交版本库） |
| `config/wopSdkConfigDefault.json` | SDK 仓库附带模板（占位符）〔目标：随仓库交付〕 |

推荐部署（K6）：正式配置**外置**于应用 jar 之外（挂载/复制），经 `WOP_SDK_CONFIG` 或 `-Dwop.sdk.config.file` 指向。**不建议**将含密钥的正式配置置于 `src/main/resources` 打入 jar——密钥会进入构建制品与 CI 缓存。

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

JSON 字段名与 Java 模型均使用 **camelCase**。

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

> v0.1 字段名 `preferredServerRoots` 改为 `backupServerRoots`（K7）：语义为「主网关失败后依序尝试的备用」，"preferred" 误导为优先于主网关。字段尚未交付，改名零成本。

#### HTTP 客户端（`httpClient` 对象）

| 字段 | 类型 | 必填 | 默认 | 说明 |
|------|------|------|------|------|
| `connectTimeout` | int (ms) | 否 | `10000` | TCP 连接超时 |
| `readTimeout` | int (ms) | 否 | `30000` | 读响应超时 |
| `maxRetryCount` | int | 否 | `3` | 跨域名重试上限；仅全局 |

### 3.4 加载校验

加载完成后 fail-fast；错误消息中的字段名与 JSON 一致（camelCase），异常统一 `WopError.configuration`：

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

## 4. 配置加载

### 4.1 `WopSdkConfigLoader`〔目标〕

SDK 配置加载入口，全 `static`，线程安全。

```java
package com.wanlianyida.wop.config;

public final class WopSdkConfigLoader {

    public static final String CONFIG_FILE_PROPERTY = "wop.sdk.config.file";
    public static final String CONFIG_FILE_ENV = "WOP_SDK_CONFIG";

    /** 按 §4.2 自动发现并加载；同一位置缓存解析结果 */
    public static WopSdkConfig loadDefault();

    /** 显式位置：文件路径或 file: URI；classpath 资源须显式加 classpath: 前缀 */
    public static WopSdkConfig load(String location);

    /** 显式文件系统 Path */
    public static WopSdkConfig load(java.nio.file.Path path);

    /** 清除加载缓存（测试 / 配置轮换编排后使用） */
    public static void clearCache();
}
```

- `load(String)` 语义无歧义（K14 附则）：带 `classpath:` 前缀走 classpath，其余一律文件系统路径；不做「先文件后 classpath」双猜。
- **缓存**：以规范化后的资源位置为 key，同一位置多次加载返回同一 `WopSdkConfig` 实例（不可变）。**无自动失效**：密钥轮换 = 外部编排调用 `clearCache()` + `WopClient.resetDefault()`，或重启（K13）。

### 4.2 自动发现顺序（`loadDefault`，K6）

命中第一个**存在且可读**的来源即加载：

| 优先级 | 来源 |
|--------|------|
| 1 | JVM 系统属性 `wop.sdk.config.file` |
| 2 | 环境变量 `WOP_SDK_CONFIG` |
| 3 | 文件 `{user.dir}/config/wopSdkConfig.json` |
| 4 | 文件 `{user.dir}/wopSdkConfig.json` |
| 5 | 文件 `{user.home}/.wop/wopSdkConfig.json` |
| 6 | Classpath `config/wopSdkConfig.json`（兜底） |

> 与 v0.1 相比 classpath 从第 3 位降为**兜底**（K6）：外部配置必须能覆盖打包默认，否则任意传递依赖 jar 携带同名资源即静默劫持运行时配置。v0.1 的 `src/main/resources` 打包推荐随之废止（§3.1）。

Classpath 读取：`Thread.currentThread().getContextClassLoader()` 优先，回退至 Loader 类 ClassLoader。

系统属性/环境变量**已设置但指向的文件不可读** → 立即 `WopError.configuration`（显式指定不容忍），**不**继续后续候选。全部来源未命中 → `WopError.configuration`，消息列出各候选实际展开路径。

### 4.3 解析管道

```
定位资源
  → UTF-8 读入（容忍并剥离 BOM；空文件报错）
  → 内置极简解析器（§4.4，未知 JSON 字段忽略）
  → 字段归一化（trim、serverRoot 去尾 /、缺省填默认值）
  → §3.4 语义校验
  → 返回不可变 WopSdkConfig
```

### 4.4 内置极简解析器（K8）

实现约束（规范性）：

- 支持类型：对象、字符串、数字（long/int 范围）、布尔、null、字符串数组、**一层**嵌套对象（`httpClient`）；更深嵌套 → configuration 错误；
- 字符串完整 RFC 8259 转义集（含 `\uXXXX`），字符串内容中的 `}` `]` `,` 不参与边界判定（D3 纪律，同 `EncryptedEnvelope`）；
- 未知字段忽略；重复键、类型不符、数字越界 → configuration 错误（消息含字段名）；
- 解析器为 core 内部类（不公开），以黄金样本 + 边界用例回归（§13）。

### 4.5 `WopSdkConfig`

不可变配置快照，持有 §3.3 全部字段。

```java
public final class WopSdkConfig {
    public String appKey();
    public String suite();
    public String merchantPrivateKey();
    public String platformPublicKey();
    public long expiredSeconds();
    public String serverRoot();
    public java.util.List<String> backupServerRoots();
    public HttpClientConfig httpClient();
    /** toString 对密钥打码（§10） */
}
```

`HttpClientConfig` 为超时与 `maxRetryCount` 的不可变值对象。

---

## 5. 一站式入口 `WopClient`〔目标，K17〕

不引入独立门面类：`WopClient` 既是协议客户端，也是一站式入口——商户面对一个类、一条路径。门面职责（懒加载、缓存复用、传输装配）由静态工厂承担：

```java
package com.wanlianyida.wop;

public final class WopClient {

    /** 惰性初始化：loadDefault（§4.2）→ SPI 发现传输（§2.2）→ Builder 构造；缓存复用同一实例 */
    public static WopClient defaultClient();

    /** 显式配置构造（不进默认实例缓存；transport 未注入时同样经 SPI 装配） */
    public static WopClient fromConfig(WopSdkConfig config);

    /** 丢弃默认实例缓存；配合 WopSdkConfigLoader.clearCache() 做轮换编排 */
    public static void resetDefault();
}
```

- **线程安全（K15）**：`defaultClient()` 惰性初始化经同步（holder 惯用法或等价），并发首调仅创建一个实例；
- 配置访问：`WopSdkConfigLoader.loadDefault()`（§4.1）——不另设 `config()` / `isInitialized()` 门面噪音；
- 密钥轮换：无自动热更新——`WopSdkConfigLoader.clearCache()` + `WopClient.resetDefault()` + 外部编排，或重启（K13）。

---

## 6. `WopClient` 实例面（增量演进，非重定义）

### 6.1 实例公开面 = 既有 + 新增（静态一站式入口见 §5）

```java
public final class WopClient {

    // ===== 既有〔现状〕——签名与语义不变 =====
    public static Builder builder();
    public RequestDraft buildRequest(String method, String path, byte[] body, SecurityLevel level);
    public VerifyResult verifyResponse(java.util.Map<String, String> headers, byte[] body, String requestPath);
    public VerifyResult verifyResponse(TransportResponse response, RequestDraft draft);
    public VerifyResult verifyCallback(java.util.Map<String, String> headers, byte[] body, String callbackPath);

    // ===== 新增〔目标〕=====
    /** 使用全局配置完成：签名 → HTTP（含状态拦截）→ 验签解密 */
    public VerifyResult execute(String method, String path, byte[] body, SecurityLevel level);

    /** 使用全局配置 + 请求级覆盖（§7） */
    public VerifyResult execute(String method, String path, byte[] body,
                              SecurityLevel level, WopRequestOptions options);

    /** 平台回调验签 + 凭证覆盖（多 appKey 商户，K10） */
    public VerifyResult verifyCallback(java.util.Map<String, String> headers, byte[] body,
                                       String callbackPath, WopRequestOptions options);
}
```

`execute` 语义（单笔调用链，全程同一 `WopRequestContext`）：

```
ctx = WopRequestContext.resolve(global, options)     // 内部，含复校验（§7.3）
draft = buildRequest(method, path, body, level)      // 复用既有实现；失败同步抛 WopError（既有契约）
response = transport.send(draft, ctx.toTransportCall())  // §8；非 2xx 在此拦截（§8.5）
return verifyResponse(response, draft)               // 复用既有实现
```

- Builder 构造时未注入 Transport → `execute` 抛 `WopError.configuration("未配置传输")`（分步 API 不受影响）；
- `verifyCallback` 重载仅消费 options 中**凭证字段**（suite / 双钥 / appKey），超时与域名字段忽略。

### 6.2 `Builder` 扩展（K11）

既有字段（`appKey` / `suite` / `merchantPrivateKey` / `platformPublicKey` / `expiredSeconds`，与现状逐字段一致）之外新增：

```java
public Builder serverRoot(String v);
public Builder backupServerRoots(java.util.List<String> v);
public Builder httpClient(HttpClientConfig v);
public Builder transport(Transport v);   // 缺省时：defaultClient()/fromConfig() 路径经 SPI 注入（§2.2）
```

程序化路径与 JSON 路径**等价**，`build()` 执行与 §3.4 相同的校验。

### 6.3 `WopRequestContext`（SDK 内部）

`com.wanlianyida.wop.internal` 包内 `final class`，不可变。**永不出现于任何 public 签名**（K2）——这是 v0.1 §8.1「public 接口方法携带包私有参数类型」缺陷的修正，public 传输面只出现 `TransportCall`（§8.1）。

---

## 7. 请求级覆盖

### 7.1 `WopRequestOptions`（商户可见）

```java
public final class WopRequestOptions {

    public static WopRequestOptions none();

    public static Builder builder();

    public static final class Builder {
        public Builder appKey(String v);
        public Builder suite(String v);
        public Builder merchantPrivateKey(String v);
        public Builder platformPublicKey(String v);
        /** ≤ 0 表示使用全局配置 */
        public Builder expiredSeconds(long v);
        public Builder serverRoot(String v);
        /** 毫秒；≤ 0 表示使用全局配置 */
        public Builder connectTimeout(int ms);
        /** 毫秒；≤ 0 表示使用全局配置 */
        public Builder readTimeout(int ms);
        public WopRequestOptions build();
    }
}
```

### 7.2 可覆盖范围

| 配置项 | 全局 JSON | `WopRequestOptions` |
|--------|-----------|---------------------|
| `appKey` / `suite` / 双钥 / `expiredSeconds` | ✅ | ✅ |
| `serverRoot` | ✅ | ✅（**整组替换候选，见 K3**） |
| `httpClient.connectTimeout` / `readTimeout` | ✅ | ✅（unirest 连接超时例外，见 K9） |
| `backupServerRoots` | ✅ | ❌ |
| `httpClient.maxRetryCount` | ✅ | ❌ |

多 appKey：出向经 `WopRequestOptions` 临时指定另一套凭证；入向回调经 `verifyCallback` 四参重载指定（K10）。

### 7.3 合并规则（完整版）与 resolve 复校验

| 项 | 规则 |
|----|------|
| 字符串字段（含凭证、serverRoot） | options 已设置且非空 → options，否则 global |
| 超时 | options 值 `> 0` → options，否则 global |
| `expiredSeconds` | options 值 `> 0` → options，否则 global |
| `backupServerRoots` / `maxRetryCount` | 恒 global |
| **Failover 候选** | serverRoot **被覆盖** → 候选 = `[options.serverRoot]`，本笔**关闭 Failover**（K3）；未覆盖 → `[serverRoot] ++ backupServerRoots`（去重保序） |

> K3 修正 v0.1 缺陷：请求级覆盖主网关（如指向测试环境）时若保留全局备用列表，主网关故障会把**生产凭证的签名请求漂移到生产网关**。覆盖即整组替换——候选集必须与主域名同源。

**resolve 复校验（fail-fast）**：合并后必须重新执行 §3.4 等价校验——suite 解析、双钥解析与套件族交叉校验（如 options 指定 `WOP-SM2-SM3` 而沿用全局 RSA 密钥 → 拒绝）、serverRoot URL 合法性。失败同步抛 `WopError.configuration`，消息风格与 §3.4 一致。v0.1 仅在加载期校验、resolve 期静默，属规格缺口。

### 7.4 性能

- `WopRequestOptions.none()`：复用预解析的默认 `WopRequestContext`；
- 凭证覆盖：按 `(appKey, suite, keyMaterial)` 缓存密钥解析结果。

---

## 8. HTTP 传输与 Failover

### 8.1 `Transport` 兼容扩展（K2）

```java
public interface Transport {
    /** 既有〔现状〕——签名与语义不变 */
    TransportResponse send(RequestDraft draft);

    /** 新增〔目标〕——default 方法，不破坏既有实现 */
    default TransportResponse send(RequestDraft draft, TransportCall call) {
        return send(draft);
    }
}

/** 每笔调用参数（public 不可变值对象；null / -1 = 用适配器默认） */
public final class TransportCall {
    /** 目标网关根地址；null = 适配器构造期 baseUrl */
    public String serverRoot();
    /** -1 = 默认 */
    public int connectTimeoutMillis();
    /** -1 = 默认 */
    public int readTimeoutMillis();
}
```

- 三个官方适配器实现双参方法；单参方法保持今日行为（构造期 baseUrl + 默认超时）；
- Failover 候选迭代位于 core 内部 `FailoverTransport`（持 Transport，实现双参内部路径），**不修改** public `Transport` 既有单参签名——商户自定义 Transport 与「自带栈消费 RequestDraft」路径（sdk-spec §1.1）不受影响；
- 商户注入仅实现单参的自定义 Transport → `execute` 可用（走 default 方法：适配器默认行为、无 Failover、无请求级超时），README 披露该限制。

### 8.2 超时映射与限制

| 配置项 | JdkHttpTransport | OkHttpTransport | UnirestTransport |
|--------|------------------|-----------------|------------------|
| `connectTimeout` | 每请求 `setConnectTimeout` | `client.newBuilder().connectTimeout`（克隆，不污染共享实例） | **实例级**（K9） |
| `readTimeout` | 每请求 `setReadTimeout` | `.readTimeout`（克隆） | 每请求超时（K9） |
| 商户注入实例（okhttp/unirest） | — | 尊重商户实例配置；请求级覆盖经克隆应用 | 同左 |

- **K9（Unirest 限制披露）**：Kong Unirest 连接超时为实例级配置、无请求级 API——请求级 `connectTimeout` 覆盖在 unirest 适配器**不生效**，实际取传输实例配置；`readTimeout` 请求级生效。README 与 Javadoc 必须披露。
- 默认值落地〔现状差异，行为变更〕：jdkhttp 现状**未设读超时**（无限挂起）——实现本规格时必须落地默认 30000 并加回归测试；okhttp 由 SDK 构造的默认实例统一 10000/30000。

### 8.3 Failover（P2）

```
本笔候选 = serverRoot 被覆盖 ? [options.serverRoot]
                               : [serverRoot] ++ backupServerRoots（去重、保序）
```

| 规则 | 说明 |
|------|------|
| 可重试（仅 pre-send） | `UnknownHostException`、`ConnectException`、`NoRouteToHostException`、`SocketTimeoutException` **且判定为连接阶段**——JDK 中连接/读超时同类异常，须按 message（"connect timed out"）判别，**无法判别一律不重试**（K4） |
| 不可重试 | 读超时（body 已发出，重复提交风险）、任何 HTTP 状态码（状态不触发换域）、`WopError`、其他 `WopSdkException` |
| 上限 | `min(maxRetryCount, 候选数 - 1)` |

全部候选失败 → `WopSdkException`（`全部网关地址不可用（已尝试 N 个）`，cause = 最后一次异常）。

> K4 修正 v0.1 缺陷：v0.1 可重试清单含 `SocketTimeoutException（connect timed out）` 但未规定判别方式；读超时一旦误重试，非幂等 POST（下单类）即重复提交。

### 8.4 方法与重定向矩阵

| 能力 | jdkhttp | okhttp | unirest |
|------|---------|--------|---------|
| 标准方法 | GET/HEAD/POST/PUT/DELETE/OPTIONS/TRACE〔现状〕 | 全部 | 全部 |
| PATCH 等扩展方法 | **fail fast 拒绝**〔现状收编〕 | 支持 | 支持 |
| 重定向 | 不跟随〔现状〕 | **钉死不跟随**〔现状为跟随，须改〕 | 不跟随〔现状〕 |

3xx 一律按 §8.5 非 2xx 处理。适配器不得改写已签名头的语义；`Content-Type` 非签名头，缺省 `application/octet-stream`〔现状收编〕。

### 8.5 响应状态语义（K5，新增——现状 statusCode 被丢弃）

`execute` 链路中 `TransportResponse.statusCode` 非 2xx → **不进入验签**，抛：

```java
public final class WopGatewayResponseException extends WopSdkException {
    public int statusCode();
    public byte[] body();          // 受 §8.6 限额约束的原始错误体
}
```

message 格式：`WOP 网关返回 HTTP <code>（响应体 N 字节）`——不内嵌 body 全文（防日志膨胀）；商户经 `body()` 取网关错误明细。

> v0.1 通篇未定义非 2xx 行为；现状实现丢弃 statusCode，502 的 HTML 错误页会以 MISSING_SIGN_HEADER 之类的验签类 reason 失败，误导排障。

### 8.6 响应体上限〔现状收编，规范性〕

三适配器已实现 11MB（`11 << 20`）流式读取上限：读取过程中逐块计数、超限即断流（sdk-spec 附录 D4）。本规格将其引用为规范性行为，任何新传输实现必须同构。

### 8.7 path 语法（K14，2026-09-11 改判）

`execute` / `buildRequest` 的 `path` 必须是 `/` 开头的**纯 API 路径**（如 `/gateway/order/create`），相对 serverRoot / `TransportCall.serverRoot` 解析；**不含 scheme 与域名部分**，亦不得携带 query / fragment。以下任一情形即 `WopError.configuration` fail-fast：

- 含 `?` 或 `#`（依据 sdk-spec 附录 G1：canonicalQueryString 恒空、网关签名面仅 POST，带 query 的路径属未定义协议面）；
- 含 scheme / 域名（`http://`、`https://` 等绝对 URL，或任何非 `/` 开头形式）。

改判依据（2026-09-11 用户裁决，撤销本日早先「绝对 URL 直连」裁定）：

1. **域名恒由 serverRoot 供给**：K3 下候选集同源（`[serverRoot] ++ backupServerRoots` 或请求级整组替换），绝对 URL 使生产签名请求可被发往任意主机，Failover 与同源候选语义一并失效；
2. **网关签名面**：sdk-spec 附录 G1 钉死 canonicalURI 为**网关所见的 path** 原样使用——全 URL 入参与平台签名面不一致，入向验签必然失败，绝对 URL 从来不是可用的正确用法；
3. 三适配器 `resolve()` 在 baseUrl 为空时对绝对 URL 的拼接行为属传输层实现细节，非本规格承诺面；`buildRequest` / `execute` 层实现 path 校验后绝对 URL 一律拒绝（§12 台账第 17 行，P0 随 execute 落地）。

---

## 9. 错误契约

| 场景 | 异常 / 结果 | 类别（sdk-spec §2.2） |
|------|-------------|----------------------|
| 配置加载 / 校验 / resolve 复校验失败 | `WopError` | configuration（明确） |
| 出向构造失败（method/path/level/L2 空 body） | `WopError` | 同 `buildRequest` 既有契约（明确） |
| 非 2xx 响应 | `WopGatewayResponseException` | 系统类（明确，含状态码） |
| 全部网关不可达 / 单笔传输失败且不再重试 | `WopSdkException` | 系统类（明确） |
| 验签 / 解密失败 | `VerifyResult.ok() == false` | I7 模糊（reason 闭集，见 sdk-spec §2.2） |

---

## 10. 安全

| 要求 | 说明 |
|------|------|
| 密钥不落日志 | `WopSdkConfig.toString` 打码；异常、debug 日志同纪律〔目标〕 |
| K16 修复锚 | 现有 `WopClient.Config.toString()` **明文打印 merchantPrivateKey**（现状缺陷）——实现本规格时必须一并修复并加回归测试（§13.9） |
| 文件部署 | 正式配置外置（K6），生产建议 `chmod 600`；不得提交版本库 |
| 回调时间窗 | 签名覆盖 `x-wop-timestamp`，**新鲜度校验不在 SDK 职责**（K10 注），由商户业务层判定 |

---

## 11. 商户接入（目标态示例）

### 11.1 Maven

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

（可选 okhttp / unirest 适配器；unirest 需商户自带 `com.konghq:unirest-java-core`，运行时 Java 11+。多适配器并存默认 jdkhttp；jdkhttp 不在 classpath 时须显式 `-Dwop.transport=` 指定，见 §2.2。）

### 11.2 配置

外置配置文件 + 环境变量 / JVM 属性指向挂载路径：

```bash
export WOP_SDK_CONFIG=/etc/wop/wopSdkConfig.json
# 或
java -Dwop.sdk.config.file=/etc/wop/wopSdkConfig.json -jar app.jar
```

### 11.3 调用

```java
import com.wanlianyida.wop.*;
import com.wanlianyida.wop.config.WopRequestOptions;

import java.nio.charset.StandardCharsets;
import java.util.Map;

public class OrderService {

    public String createOrder() {
        byte[] body = "{\"orderId\":\"W1\"}".getBytes(StandardCharsets.UTF_8);

        VerifyResult result = WopClient.defaultClient().execute(
                "POST", "/gateway/order/create", body, SecurityLevel.L0);

        if (!result.ok()) {
            throw new IllegalStateException(result.message());
        }
        return result.plaintextAsUtf8();
    }

    public String createOrderWithLongTimeout(byte[] body) {
        VerifyResult result = WopClient.defaultClient().execute(
                "POST", "/gateway/big/query", body, SecurityLevel.L2,
                WopRequestOptions.builder().readTimeout(120_000).build());
        if (!result.ok()) {
            throw new IllegalStateException(result.message());
        }
        return result.plaintextAsUtf8();
    }

    /** 多 appKey：回调验签时指定该 appKey 的凭证 */
    public VerifyResult handleCallback(Map<String, String> headers, byte[] rawBody) {
        return WopClient.defaultClient().verifyCallback(headers, rawBody, "/merchant/callback",
                WopRequestOptions.builder()
                        .appKey("app_002")
                        .platformPublicKey(PLATFORM_PUB_002)
                        .build());
    }
}
```

### 11.4 Spring Boot

```java
@Configuration
public class WopConfiguration {

    @Bean
    WopClient wopClient() {
        return WopClient.defaultClient();
    }
}
```

容器启动时创建 Bean → 触发配置加载；配置错误在**启动阶段**暴露。

---

## 12. 实现差距台账（现状 → 动作）

| # | 能力 | 现状 | 动作 | 阶段 |
|---|------|------|------|------|
| 1 | 配置加载 / 校验 / 仓库模板 | 无 | 新增（内置极简解析器，§4.4） | P0 |
| 2 | `WopClient` 静态一站式入口（defaultClient / fromConfig / resetDefault） | 无（仅有 `builder()` 实例路径） | 新增（线程安全惰性初始化，K17/K15） | P0 |
| 3 | `execute` ×2 | 无（`WopClient` 无 Transport 字段） | 新增，组合既有 buildRequest/send/verifyResponse | P0 |
| 4 | `WopClient.Config.toString` 打码 | 明文打印私钥（缺陷） | 修复 + 回归测试 | P0 |
| 5 | `WopRequestOptions` / `WopRequestContext` | 无 | 新增（Context 包私有，不出 public 签名） | P1 |
| 6 | `verifyCallback` 四参重载 | 无 | 新增 | P1 |
| 7 | `Transport.send` 双参 default + `TransportCall` | 无（单参为现状） | 增量扩展，不动既有签名 | P1 |
| 8 | 三适配器双参实现 | 仅单参 | 实现（unirest 连接超时按 K9 披露限制） | P1 |
| 9 | jdkhttp 读超时默认 30000 | 未设（无限挂起） | 修改（行为变更 + 回归） | P1 |
| 10 | okhttp `followRedirects(false)` | 跟随（与 jdkhttp 不一致） | 修改 | P1 |
| 11 | 非 2xx 拦截 + `WopGatewayResponseException` | statusCode 被丢弃 | 新增 | P1 |
| 12 | resolve 复校验 | 无（v0.1 亦未规定） | 新增（§7.3） | P1 |
| 13 | `FailoverTransport` | 无 | 新增 | P2 |
| 14 | `TransportFactory` 接口 + 三适配器 `META-INF/services` 注册 | 无 | 新增（core 无环构造默认传输的唯一路径，K18） | P0 |
| 15 | `wop.transport` 属性 + 多 factory 默认选择 | 无 | 新增（非法值/歧义 fail-fast） | P2 |
| 16 | D4 11MB 上限 / 方法白名单 / octet-stream | 已实现 | 收编为规范（无代码动作） | — |
| 17 | path 校验（§8.7：buildRequest 与 execute 两层，纯 API 路径） | 无（现状不校验 path） | 新增（scheme/域名/`?`/`#` 拒绝，WopError.configuration） | P0 |

阶段划分：**P0** = 最小可用一站式路径（SPI 发现唯一传输、无 Failover、候选仅 serverRoot）；**P1** = 请求级覆盖 + 状态语义 + 适配器补齐；**P2** = Failover 与多传输选择。

---

## 13. 测试要点

1. 发现顺序六来源各路径 + classpath 兜底 + 显式指向不可读即报错 + 全未命中错误消息；SPI：恰一 factory 即用、零/多 fail-fast（P2：多 factory 默认与 `wop.transport` 指定/非法值/jdkhttp 缺席须显式）
2. 加载缓存 / `clearCache()` / `WopClient.resetDefault()`；同位置同实例；`defaultClient()` 并发首调单实例（K15）
3. 加载校验全表（含重复键、类型不符、数字越界、BOM、空文件）
4. 极简解析器黄金样本 + 转义/边界用例（D3 纪律）
5. resolve 合并与复校验：套件族与密钥交叉拒绝、`expiredSeconds ≤ 0` 回退、serverRoot 覆盖关闭 Failover
6. 非 2xx（含 3xx）不进验签；`WopGatewayResponseException.statusCode()/body()` 可达
7. Failover：pre-send 异常可重试、读超时/HTTP 状态不重试、`min(maxRetryCount, 候选-1)`、全候选失败消息
8. 方法矩阵（jdkhttp PATCH 拒绝；三适配器均不跟随重定向）
9. 密钥不出现在日志与 `toString`（含 K16 回归：`WopClient.Config.toString` 打码）
10. path 语法：仅 `/` 开头纯 API 路径；含 scheme/域名（绝对 URL）、`?`、`#` 一律 `WopError.configuration` 拒绝
11. Gherkin（sdk-spec E3）：配置加载、execute L0/L2、回调验签场景 ≥10；平台响应构造遵守 D5（不复用被测出向代码）

---

## 14. 决策记录（K 系列）

| # | 决策 | 依据 |
|---|------|------|
| K1 | 文档性质改为**目标态规格 + 现状锚点 + 差距台账**，废止 v0.1 的现在时描述 | v0.1 以现在时描述零实现且从未存在的机器（git 全历史无门面提交） |
| K2 | `Transport` 既有单参签名不动；新增 default 双参方法 + public `TransportCall`；`WopRequestContext` 永不出现在 public 签名 | v0.1 §8.1 public 接口方法携带包私有参数类型，包外不可实现；且静默破坏三适配器与商户自定义 Transport |
| K3 | 请求级 serverRoot 覆盖 = 候选集整组替换，本笔关闭 Failover | v0.1 保留全局备用列表 → 跨环境凭证漂移（生产签名请求可被发往生产备用网关） |
| K4 | 重试仅限 pre-send 失败；SocketTimeout 须按 message 判别连接阶段，不可判别不重试 | JDK 连接/读超时同类异常；读超时重试 = 非幂等 POST 重复提交 |
| K5 | 非 2xx 不进验签，抛 `WopGatewayResponseException`（statusCode + body 访问器） | v0.1 未定义；现状丢弃 statusCode，错误页以验签类 reason 失败误导排障 |
| K6 | 发现顺序外部优先（sysprop > env > user.dir > user.home > classpath 兜底）；正式配置外置 | v0.1 classpath 第 3 位 + resources 打包推荐 = 传递 jar 劫持 + 密钥入制品 |
| K7 | `preferredServerRoots` 改名 `backupServerRoots` | 语义为有序备用，"preferred" 误导为主优先；字段未交付，改名零成本 |
| K8 | 配置解析用内置极简读取器；jackson 维持 test scope，不进运行时依赖面 | v0.1 声称 jackson「随 core 传递」与 pom 事实（test scope）不符；10 个扁平字段不值 databind 依赖面；core 已有 D3 解析纪律先例 |
| K9 | unirest 连接超时为实例级（无请求级 API），请求级覆盖不生效，须披露 | Kong Unirest 能力边界；诚实披露优于不可实现的表格 |
| K10 | `verifyCallback` 增加四参凭证覆盖重载；时间窗校验归属商户业务层 | v0.1 多 appKey 仅出向方案，入向无法验第二 appKey；SDK 层无重放窗职责划分 |
| K11 | `Builder` 扩展全字段 + 可选 Transport，程序化与 JSON 等价 | v0.1 Builder 无 serverRoot，构造的 client 无网关地址 |
| K12 | `wop.transport` 非法值 fail-fast 列出可用 factory，禁止静默回退 | 静默回退掩盖装配错误 |
| K13 | 配置缓存无自动失效；轮换 = `clearCache()` + `WopClient.resetDefault()` + 外部编排或重启 | 显式决策，避免「缓存即热更新」误读 |
| K14 | `load(location)` 语义钉死（classpath: 前缀显式区分）；`path` = `/` 开头**纯 API 路径**（不含 scheme/域名；`?`/`#`/绝对 URL 一律拒绝） | 消除 v0.1 双猜歧义；G1 钉死 canonicalURI 为网关所见 path，全 URL 与平台签名面不一致；域名恒由 serverRoot 供给（K3 同源候选与 Failover 语义）——2026-09-11 用户改判，撤销同日早先允许绝对 URL 直连的裁定 |
| K15 | `WopClient.defaultClient()` 线程安全钉死（并发首调单实例） | v0.1 未规定 |
| K16 | `WopClient.Config.toString()` 私钥打码为 P0 修复锚 | 审查发现的现存泄漏，随本规格落地一并修复 |
| K17 | 不引入独立 `WopSdk` 门面；`WopClient` 即一站式入口（静态 `defaultClient()` / `fromConfig()` / `resetDefault()`） | 两个公开入口（持凭证与传输的 `WopClient` vs 薄包装 `WopSdk`）重叠徒增困惑与文档面；门面职责全部可由静态工厂承担（2026-09-11 用户裁决） |
| K18 | `TransportFactory` SPI 前移 P0（接口在 core + 三适配器 META-INF/services 注册）；P0 规则 = 恰一 factory 即用，零/多 fail-fast；多 factory 默认与 `wop.transport` 选择留 P2；jdkhttp 缺席时禁止「取首个」、须显式指定 | 依赖方向 适配器 → core，「P0 固定 jdkhttp」需 core 构造 `JdkHttpTransport`，唯一手段是循环依赖或反射；ServiceLoader 枚举序不稳定（2026-09-11 评审意见） |
