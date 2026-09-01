# WOP 商户 SDK 统一规格 Spec

> 版本：v1.0-ratified
> 日期：2026-08-28
> 裁决记录：Q1 传输层=协议核心+可插拔 HTTP 适配层（用户裁决）；Q7 TS/PHP 首版仅 RSA 套件、国密列路线图（用户裁决）；Q2–Q6 按草案默认立场通过（已列明无异议）
> 适用仓库：github.com/wop-platform/wop-{lang}-sdk
> 公开真源：[wop-platform/wop-specs · sdk/wop-sdk-spec.md](https://github.com/wop-platform/wop-specs/blob/main/sdk/wop-sdk-spec.md)——即本文件（wop-specs 仓唯一维护版）；网关仓保留同名工作副本，修订以本仓为准并双向同步

---

## 1. 定位与范围

商户侧接入 WOP 网关的官方客户端库：封装协议核心（签名/摘要/数字信封/验签解密），
使商户无需理解 canonicalRequest、套件推导与线上字节格式即可安全对接。

### 1.1 架构（Q1 定稿）

- **协议核心（必做，98% 门禁对象）**：纯函数产出 `(headers, wireBody)` 与校验/解密入口，零网络 IO、零第三方运行时依赖
- **HTTP 适配层（必做，同门禁）**：`Transport` 抽象 + 各语言主流客户端适配器，适配器以独立模块/peer 依赖交付（不污染核心依赖面）；商户自带栈时直接消费 RequestDraft

| 语言 | 适配器交付 |
|------|-----------|
| Java | `wop-sdk-core` + `wop-sdk-okhttp`（okhttp provided）+ `wop-sdk-jdkhttp`（java.net.http，零依赖） |
| Go | `Transport` 接口 + 默认 `http.Client` 实现 + `RoundTripper` 桥接 |
| TypeScript | fetch 原生适配器 + axios peer 适配器 |
| Python | stdlib urllib 适配器 + httpx/requests peer 适配器 |
| PHP | curl 扩展适配器 + Guzzle peer 适配器 |
| .NET | `HttpClient` 适配器（DelegatingHandler 可插拔） |

### 1.2 套件支持矩阵（Q7 定稿）

| 语言 | RSA3072/4096 | SM2-SM3 |
|------|---------------|---------|
| Java / Go / Python / .NET | ✅ | ✅（BC / emmansun / gmssl / BC.Net） |
| TypeScript / PHP | ✅ | ❌ 首版明确抛“暂未支持”错误 + README 路线图（向量 fixture 全量拷贝，SM 段不消费，另有“SM 套件必须拒”的负测试） |

### 1.3 功能面

| # | 功能 | 说明 | 依据 |
|---|------|------|------|
| F1 | 套件配置与解析 | securityReq 三套件（RSA3072/4096、SM2-SM3），跨族/非法拒绝 | spec §2 |
| F2 | canonicalRequest 构造 | 5 段 `\n`；header 值 Java-URLEncoder 语义（空格→%20 等） | §1.3 排除项→SDK 承接 |
| F3 | 结构化 x-wop-sign | 商户私钥加签（出向）/ 平台公钥验签（响应与回调） | §3.3①、§7.3 |
| F4 | x-wop-content-digest | `alg 小写hex` 恰一空格；算法随套件族；无 body 缺席；有 body 必传必入签 | D2/D3/I1 |
| F5 | L2 数字信封 | AES-256-GCM / SM4-GCM 全文加密；DEK 载荷 `alg$key$iv`；RSA-OAEP（显式双 SHA-256+空 label）/ SM2(C1C3C2) 包装 | §3.3②③、§6 |
| F6 | 响应/回调校验 | 验签→digest 复核→DEK 解包→alg 族比对→bulk 解密，顺序固定 | I2/I3 |
| F7 | 线上字节格式 | base64url 无填充（拒收 `=`）；SM2 签名裸 r‖s 64B；SM2 密文 C1C3C2；RSA 公钥 SPKI/SM2 未压缩点 | §3.3/§3.4/D9/D10 |
| F8 | 向量合规 | 测试消费黄金向量 fixture，**字节级**一致；负向量（tamper/跨族/错格式）必须拒 | D9/B.2 |
| F9 | 防重放辅助 | CSPRNG nonce 生成、毫秒时间戳、expiredSeconds 组装 | §7 |

## 2. 概念 API（各语言惯用映射）

```
WopClient / WopConfig
  ├─ config: appKey, suite(securityReq), merchantPrivateKey, platformPublicKey, [gatewayBaseUrl]
  ├─ buildRequest(method, path, body?, level=L0/L2) → RequestDraft{headers, wireBody}
  ├─ verifyResponse(headers, body) → VerifyResult{ok, plaintext?, reason?}   # 模糊 reason（I7）
  ├─ verifyCallback(headers, body, callbackPath) → VerifyResult              # URI 取回调 path
  └─ errors: 配置/协议类错误明确；验签/解密失败对外模糊（I7 纪律）
```

- 密钥入参：字符串（PEM 或 Base64 单行），SDK 内部解析；RSA=SPKI/PKCS8、SM2=04‖X‖Y/d 标量（D12）
- 确定性要求：同输入同输出（除 CSPRNG IV/nonce）；`buildRequest` 可重放生成（幂等测试断言）

### 2.1 出向必传 header 契约（必传集合与入签义务，2026-08-31 增补）

`buildRequest` 产出的 `RequestDraft.headers` 必须包含下表集合，且**全部参与签名**
（canonicalRequest 的 signedHeaders 集合；I1「digest 必入签」为本表子集不变式，此处钉死全集）。
缺失任一必传项视为非法输入，构造即拒（`configuration` 类，见 §2.2），禁止静默补默认值：

| header | 必传条件 | 参与签名 | 值来源 |
|--------|----------|----------|--------|
| `x-wop-appkey` | 恒必传 | ✅ | `config.appKey` 同值序列化；SM2-SM3 套件下 ZA 计算的 userId 即此值（D14） |
| `x-wop-content-digest` | 有 body 必传；无 body（GET）缺席——不定义空摘要中间态（D2） | ✅ | wire body 原始字节摘要，`alg 小写hex`（F4） |
| `x-wop-encrypt` | 仅 L2 加密请求必传；L0 缺席 | ✅ | DEK 载荷 `L2;dek=alg$key$iv`（F5） |
| `x-wop-nonce` | 恒必传 | ✅ | CSPRNG（F9） |
| `x-wop-timestamp` | 恒必传 | ✅ | 毫秒时间戳（F9） |

- `x-wop-sign`（securityReq 声明与签名的载体）不参与自身签名。
- SM2-SM3 下 ZA 的 userId 取**出向请求实际携带的 `x-wop-appkey` 值**（= `config.appKey`
  的同一序列化结果），闭环 D14：签名身份与请求身份恒同源，禁止读库默认值。

### 2.2 错误契约（WopError，2026-08-31 增补）

出向可观测错误（构造器 / `buildRequest` 同步 throw、async reject）统一形状
`WopError{category, message}`；`category` 为闭集（小写 ASCII，跨语言恒定），映射与 I7 文案纪律：

| category | 触发类 | I7 文案 |
|----------|--------|---------|
| `configuration` | 配置错误：appKey / 密钥材料缺失或非法、securityReq 非法或跨族（F1） | 明确 |
| `parse` | 协议解析错误：header / 信封 / 线上编码格式（D1/D3） | 明确 |
| `unsupported` | 能力不支持：合法套件但本 SDK 未实现（如 TS/PHP 首版 SM，§1.2） | 明确 |
| `integrity` | 完整性校验：digest 不匹配 | 明确 |
| `consistency` | 一致性校验：dek alg 与套件族不符（I3） | 明确 |
| `signature` | 验签失败 | **模糊**：固定文案「签名验证失败」 |
| `decrypt` | 解密失败（DEK 解包 / GCM tag 两路径同文案，防 oracle 区分） | **模糊**：固定文案「解密失败」 |

- 入向校验（`verifyResponse` / `verifyCallback`）不抛 WopError，吞并为
  `VerifyResult{ok, reason}`（I7）；该路径 category 不可观测（D6 禁伪断言的依据）。
- 错误类目跨域对照（本表七值 ↔ crypto-spec §10.2 网关九类 ↔ interop canonical 五类 ↔ playbook 伪码）见 crypto-strategy-spec §10.3。

## 3. 各语言实现约束

| 语言 | 仓库名 | 密码依赖（唯一指定路径，E5） | 测试/覆盖率工具 |
|------|--------|------------------------------|------------------|
| Java | wop-java-sdk | JCA + BouncyCastle（SM 全覆盖） | JUnit5 + JaCoCo（line+branch） |
| Go | wop-go-sdk | crypto/rsa、crypto/cipher + emmansun/gmsm | go test -covermode=atomic + 强制分支清单 |
| TypeScript | wop-typescript-sdk | WebCrypto（Node ≥18 / 浏览器）+ 纯 TS SM2/SM3/SM4-GCM | vitest + coverage-v8（branch） |
| Python | wop-python-sdk | cryptography（RSA/AES）+ gmssl（SM） | pytest + coverage.py（branch） |
| PHP | wop-php-sdk | phpseclib≥3（RSA/OAEP）+ 纯 PHP SM2/SM3/SM4-GCM | PHPUnit + xdebug `--path-coverage`（branch，唯一驱动） |
| .NET | wop-dotnet-sdk | BouncyCastle.Net（全量，含 SM） | xUnit + coverlet（branch） |

- TS/PHP 的纯实现 SM 系算法：以黄金向量为唯一正确性锚（gateway spec D11：官方 SDK 即 SM 生态答案）
- Go 分支覆盖：语句覆盖 ≥98% + 显式分支矩阵测试（Go 原生不产分支计数，spec 验收按语句+负向量清单）
- PHP 分支覆盖（wop-php-sdk 实证钉死，2026-08-29）：pcov 无分支数据、phpdbg 驱动已被 PHPUnit 10+ 移除，均为死路——行+分支双门禁唯一驱动是 xdebug path coverage（本地 `XDEBUG_MODE=coverage php -d memory_limit=2G vendor/bin/phpunit --path-coverage …`）；CI 用 shivammathur/setup-php 的 `coverage: xdebug` 输入安装，禁止 `pecl install xdebug || true` 吞错（runner php_dir 不可写必失败，CI run 33196709945 教训）；分支门禁解析 `--coverage-php` 快照（XML 报告只有行数据）；PHP ≥8.5 下命名空间内裸全局函数调用会产生 frameless-call 双路径、Xdebug 将 NS 慢路径块记为未覆盖分支，src 全局函数调用一律 `\` 前缀

## 4. 仓库与工程约定

- 组织 `wop-platform`，仓库 `wop-<lang>-sdk`，主分支 `main`，MIT License，版本 0.1.0
- 目录：`src/`（或语言习惯）、`tests/`、`vectors/crypto-vectors.json`（fixture 副本，禁止手改；公开真源：[wop-specs · crypto/crypto-vectors.json](https://github.com/wop-platform/wop-specs/blob/main/crypto/crypto-vectors.json)）
- `README.md`（**中文默认**）+ `README.en.md`（英文），内容含：快速开始、密钥准备、L0/L2 示例、向量自测、错误处理与模糊化说明
- CI（GitHub Actions）：测试 + 覆盖率门禁（≥98%）+ 向量合规必须全绿
- 提交规范沿用 conventional commits（中文 body 允许）

## 5. 验收标准（每仓通用，CI 强制）

| # | 验收项 | 判定 |
|---|--------|------|
| A1 | 黄金向量正向量 | 每条字节级一致（签名/密文/摘要/DEK 组装） |
| A2 | 黄金向量负向量 | tamper/跨族/错长度/带 `=` base64 全部拒绝 |
| A3 | 行覆盖率 | ≥98%（变更/全部源码，以各语言工具为准） |
| A4 | 分支覆盖率 | ≥98%（Go 按 §3 约定替代口径） |
| A5 | 双语 README | 中文默认 + 英文，含四段必备（快速开始/密钥/L0L2/向量自测） |
| A6 | 协议语义 | D2 无 body 缺席、I1 digest 入签、I7 错误模糊、F6 校验顺序 |
| A7 | 构建 | 语言标准构建零警告级错误，CI 绿 |

## 6. 决策记录

Q1/Q7 用户裁决，Q2–Q6 默认通过（ratify 无逐条分歧记录，以草案默认立场整体通过），详见文件头。spec 冻结为 v1.0-ratified。

## 附录 D：跨语言实现勘误与补充纪律（2026-08-29 增补，v1.0-ratified 后勘误）

> 背景：wop-dotnet-sdk 交付审查发现 base64url 非规范尾随位在 .NET/JDK/CPython/PHP 标准库中均为宽容实现，
> 已升格为 spec 层测试向量（crypto-vectors.json formatRules 8→12，gateway commit 18836a2），
> 并完成六仓横审与修复。本附录条款与正文同级生效。

### D1. base64url 尾随位严格性（F6/F7 细化）

- 未填充 base64url 的**非规范尾随位必须拒绝**（语义锚：Go `base64.RawURLEncoding.Strict()`，RFC 4648 §3.5）：
  - `len % 4 == 2`（1 字节数据）：末字符 index 低 **4** 位须为零；
  - `len % 4 == 3`（2 字节数据）：末字符 index 低 **2** 位须为零；
  - `len % 4 == 1` 一律拒绝。
- 字符 index：A-Z=0-25、a-z=26-51、0-9=52-61、`-`=62、`_`=63。
- 黄金向量：`aE`/`TWF` → reject；`AA` → accept（0x00）；`TWE` → accept（`"Ma"`）。
- 实现注意：.NET `Convert.FromBase64String`、JDK `Base64.getUrlDecoder()`、CPython `base64.urlsafe_b64decode`、
  PHP `base64_decode($s, true)` 均不校验尾随位，**必须在解码前显式校验**。

### D2. 向量 fixture 同步机制（A1/A2 执行细则）

- 各仓 fixture 是真源（本仓 `crypto/crypto-vectors.json`，wop-specs）的**字节副本**；CI 必须含"真源 vs 本地副本
  字节比对"步骤，**不一致即 fail**（禁止降级为 warning）。
- formatRules 消费**三件套**（缺一即视为未消费）：
  1. 循环全量消费（禁止按 id 点名）；
  2. 未知 id 哨兵（出现未预期条目即失败）；
  3. 条数哨兵（`assertEquals(<真源当前条数>)`，真源条数变更时各仓哨兵必须同步更新）。
- accept 向量必须有正向断言（解码字节级），禁止只测 reject 路径。

### D3. 信封 JSON 解析（F5/L2 细化）

- `{"encrypted":...}` 提取必须容忍未知字段，且解析器必须**字符串感知**（字符串内容中的
  `}` `]` `,` 不得参与深度/边界判定）。
- 手写扫描器必须支持 RFC 8259 完整转义集（含 `\uXXXX`；键名转义与其解码字符语义等价）；
  推荐直接使用语言标准 JSON 解析器。

### D4. 传输层限额（D5 执行细则）

- 响应体上限 11MB（`11 << 20` 字节）级，必须在**读取过程中生效**（流式累计、超限即断流），
  禁止整体缓冲后才检查、禁止依赖运行时默认整体缓冲架空自定义限额。

### D5. 测试构造纪律

- 入向校验测试的"平台响应"构造**不得复用被测 SDK 的出向代码**（防镜像偏差：
  出向/入向共用同一实现时，协议理解错误双向对称而测试全绿——见 wop-php-sdk L2 裸密文事故）。

### D6. 错误断言纪律（A6/I7 执行细则，2026-08-29 wop-typescript-sdk 变异闭环实证后增补）

> 实证背景：SDK 测试四维 100% 且 146 测试全绿时，自研变异运行（263 变异体、12 类算子）
> 首轮击杀率仅 76.01%——最大漏洞是错误文案断言依赖框架子串语义（如 vitest/Jest 的
> `toThrowError(string)` 为 **contains** 匹配），对「错误消息回归」与字符串追加类变异体
> 双重失明；补全等断言后击杀率升至 95.06%。覆盖率门禁测不到这一层。

- **协议类错误**（parse / unsupported / integrity / consistency，即 I7 语义明确类）：
  测试必须断言错误消息**全等**（`expect(e.message).toBe(exact)` 或等价机制），
  禁止以框架 `toThrowError(string)` / `toContain` / 无 `$` 锚正则作为唯一文案断言
  ——子串语义对文案尾部漂移天然放行。
- **密钥参与类错误**（signature / decrypt，I7 模糊类）：只断言**固定模糊文案本身**
  （如「签名验证失败」「解密失败」），且断言不匹配实现细节（padding/tag/密钥不符等）；
  两条失败路径（DEK 解包失败与 GCM tag 失败）必须同文案，防 oracle 区分。
- **category 断言**：错误对象可到达的路径（构造器/出向 API 的同步 throw、async reject）
  必须显式断言 `category` 全等（闭集与映射见 §2.2，不得自造取值）；入向校验把 WopError
  吞并为 `VerifyResult{ok, reason}` 的路径上 category 不可观测，禁止为其编写伪断言
  （应归类为等价变异体，见下条）。
- **变异测试幸存体处置**：每轮变异运行的幸存体必须**逐个归档**——附等价性证明
  （黑盒不可区分的构造性论证）或测试缺口修复，禁止只报击杀率数字；
  环境等价（仅特定运行时可区分）须注明触发环境（范例：wop-typescript-sdk
  `tests/mutation/EQUIVALENT-MUTANTS.md`，13 个幸存体全部举证）。
- 等价变异体中的**死代码信号**（如套件缓存表预注册使支持表对应项黑盒不可达）应回溯
  main 侧简化建议，经 PR 处理；测试侧不得以「测不可达代码」的方式制造假覆盖。

### D7. 变异测试门禁档位（2026-08-31 wop-dotnet-sdk 变异闭环实证后增补，已经 PR #6 评审合入）

> 实证背景：wop-dotnet-sdk 四维覆盖率 100% 后，Stryker.NET 4.16（Advanced 级、1194
> 变异体、24 类算子）首轮击杀率仅 83.17%——行/分支 100% 覆盖对「错误文案漂移、解码
> 索引边界、随机流合同、时间窗上界」类回归完全失明；四轮补测后 96.5%（存活 29 个
> 逐个论证等价）。与 wop-typescript-sdk 自研变异闭环（D6 实证）互为佐证：
> 覆盖率门禁之上的变异档位是必要的第二层防线。

- **门禁档位**：击杀率（killed+timeout / 有效且非等价变异体）**≥ 90%**；编译无效/构建错误、
  无覆盖及其他未执行变异体**不计入分母**（须单独统计并归档，见「分母披露」），与 A3/A4
  覆盖率门禁（≥98%）并列；作为定期/发布前档位运行（变异全量跑进每 PR 的成本由各仓自评），
  结果归档进仓内 `StrykerOutput` 等价物或 `tests/mutation/` 目录。
- **算子多样性**：受测变异体须覆盖 ≥10 类算子，且至少含条件边界、数学、返回值/语句、
  常量（字符串/布尔）四族——单一算子族的高击杀率不构成通过依据。
- **等价清单评审**：从分母剔除等价变异体须满足 D6 幸存体处置条款（逐个举证），
  且排除配置（行级/字符跨度级）与举证文档一并入 PR 评审——排除项是审查重点，
  禁止「先排除后论证」。
- **分母披露**：门禁报告必须同时给出四个计数——生成总数、无效数（编译/构建错误）、
  等价排除数、计分数（= 生成总数 − 无效 − 等价排除）——以及原始得分
  （killed+timeout / 生成总数）与排除后得分（killed+timeout / 计分数）；仅排除后得分
  经 PR 评审（等价清单获批）后方可作为门禁依据。未披露排除规模的得分跨仓不可比，
  禁止以扩充等价清单换取达标。
- **工具口径注记**（Stryker.NET）：行级细粒度过滤语法 `File.cs{start..end}` 的
  start/end 为**字符索引**而非行号（官方文档明确），直接写行号会静默错位；
  须由脚本从行号清单换算（范例：wop-dotnet-sdk `scripts/gen-stryker-excludes.py`）。

## 附录 E：跨语言质量任务统一契约（2026-08-29 增补）

> 背景：六仓质量闭环（覆盖率 100% + 变异 + Gherkin）执行中沉淀的两条契约，供未来跨语言
> 质量任务/子代理分派时统一遵守。本附录条款与正文同级生效。

### E1. Go 覆盖率口径（禁止语句覆盖误报分支覆盖）

- `go test -cover` 仅报告**语句覆盖率**，无分支维度。**任何以"分支覆盖"名义验收 Go 的任务，
  必须使用 gcov（`-b` 分支信息）或等价分支计数工具**；工具不可用时，按 §3 约定采用
  "语句覆盖 ≥98% + 显式分支矩阵测试（负向量清单，`spec:<ID>` 注释索引）"替代口径。
- 报告中必须**显式标注口径**（"语句覆盖 X%"或"分支覆盖 X%（gcov）"），禁止裸报"覆盖 X%"
  使读者误判为分支覆盖。
- 覆盖率未达 100% 时的结构性不可达语句：须逐条论证不可达性（前置条件/类型系统/常量折叠），
  论证写入 docs/ 并附证据，不得以"未覆盖"草草带过。

### E2. 变异测试工具兜底（工具起不来 → 脚本化变异）

- 各语言变异工具（Java: PIT / Go: gremlins / Python: mutmut / TS: StrykerJS / PHP: infection /
  .NET: Stryker.NET）**大概率未预装**。分派前约定：
  1. 优先尝试官方工具（安装失败/环境不兼容即记录并降级，**不得卡在装工具上**）；
  2. 工具不可用时采用**脚本化变异**：对源码施加等价变异算子（条件边界/取反、数学运算、
     返回值、常量、逻辑、空返回等，≥10 类），逐个变异→编译→跑测试→击杀判定；
  3. 击杀率目标 **≥90%**；存活变异体须逐条判定：真实测试缺口（补测试）或行为等价
     （黑盒等价证明，记录到 docs/mutation-report.md）。
- 变异脚本自身须：变异后源码**恢复原样**（内存/文件还原 + 校验）、基线测试先绿、
  超时/崩溃算击杀（行为破坏亦是被捕获的缺陷）。

### E3. Gherkin 场景锚定规格面

- 各语言 Gherkin 框架（godog / pytest-bdd / cucumber-js / behat / SpecFlow）场景 ≥10，
  覆盖 F1-F9 + 概念 API（§2）使用场景；"平台响应"构造遵守 D5（不复用被测出向代码）。
