# WOP SDK Interop 样本集 v1（协议编排跨仓一致性合同）

> 真源：本目录（`wop-specs/interop/v1/`）。各 SDK 仓拷贝字节副本进测试 fixture（与
> `crypto/crypto-vectors.json` 同一纪律：禁手改、CI 与本地消费同一副本）。
> 生成器：`wop-go-sdk` 的 `interopgen_test.go`（`UPDATE_INTEROP=1 go test -run TestInteropGenerate`），
> 生成结果确定性（两次生成 sha256 一致）。样本密钥全部 TEST-ONLY，与黄金向量同源。
> 版本戳纪律：`_meta.specVersion` 随 spec 版本事件由**再生成**刷新（禁手改）；再生成时必须
> 补记 `generatedAt`（RFC3339）；消费仓以 `_meta` 为样本集版本的唯一依据。
> 现状注记（2026-09-01）：`_meta.specVersion` 现值为 v0.3 合并口径；v0.4-draft 变更（D14 出向钉死等）
> 不改变样本字节语义——wop-go-sdk main 上 Build/Verify 消费测试双绿（2026-09-01 M2 取证）。
> `_meta` 刷新随六仓协同再生成执行（单仓改字节会分叉冻结合同），interop 样本集此刻无需再生成。
> 现存 `_meta` 无 `generatedAt` 字段（v1 冻结早于上述纪律的 2026-09-01 增补）；消费仓当前以
> `_meta.generatedBy`（wop-go-sdk@0.1.0）与冻结合同版本为依据，该字段随下次再生成补齐。

## 目的

黄金向量证明**算法原语与线上编码**跨语言一致；本样本集补上**协议编排**一致性：
canonicalRequest 拼装、x-wop-sign 结构、signedHeaders 组装、L2 信封、F6 校验顺序
与**错误分类**（明确 vs 模糊，I7 分界）。canonicalRequest 拼装的字节级权威条文见
sdk-spec 附录 G（G1–G3，2026-09-01 立法；以本样本集与参考实现 wop-go-sdk main 为事实源）。

## 两种用例方向

### kind = build（6 条：3 套件 × L0/L2）

同 `input`（固定 timestamp/nonce/randomHex）必须复现同 draft：

- `reproduceMode: "byte-exact"`（RSA 族）：所有头与 wire body 字节级一致
  （PKCS#1 v1.5 与 OAEP-from-stream 均确定）
- `reproduceMode: "deterministic-fields"`（SM2 族）：除 `opaque` 声明的
  密钥参与段外全部字节级一致：
  - `x-wop-sign.signatureSegment`：末段 `/` 之后的签名值（k 为 CSPRNG，合法变化）
  - `x-wop-encrypt.dekValue`：`dek=` 之后的包装密文（SM2 k 同理）
  - wire body、digest 头**仍在比对范围**（CEK/IV 由随机流前段确定）

**随机流消费顺序合同**：`randomHex` 的消费顺序为
`[16B nonce 池（仅当 nonce 未注入时才消费）][CEK][12B IV][k…（各仓实现自定义）]`。
**build 样本的 nonce 均已注入**，故流偏移 0 直接是 CEK（`[0:len(CEK)]→CEK`、
随后 12B→IV、RSA 再取 32B OAEP seed）。字面读成"先跳 16B 再取 CEK"会导致比对失败。

### kind = verify-positive / verify-negative（7 + 16 条）

样本是**冻结数据**（响应头 + wire body），消费仓不得重新生成：

- positive：`VerifyResponse`/等价入口应通过，且解密明文与 `plaintextB64` 一致；
  含混合大小写头名变体（P7——Go 等以 net/http 结构性规范化满足的仓，消费端以
  小写化进入即可）
- negative：必须拒绝，且**错误分类**与 `errorClass` 逐条对账

## 错误分类合同（canonical classes）

| class | 语义（10.2） | 典型样本 |
|---|---|---|
| `verify-failed` | 验签类，**模糊**（文案恒定无细节） | n16 重放等签名层故障 |
| `decrypt-failed` | 解密类，**模糊** | n01 密文损伤、n05 C1C2C3、n13 DEK 键长 |
| `digest-mismatch` | 完整性类，明确 | n02、n09（**n09"有 body 缺 digest 头"亦归此类**——D2 把"缺失"与"不匹配"同视为完整性破坏，非结构格式问题） |
| `alg-mismatch` | 一致性类，明确（D8） | n04 |
| `protocol` | 解析/协议结构类，明确 | n03/n06/n07/n08/n10/n11/n12/n14/n15 |

**跨域对照**：canonical 五类 ↔ crypto-spec §10.2 网关九类 ↔ sdk-spec §2.2 出向七值的完整映射见 crypto-strategy-spec §10.3。

**已裁决的跨仓分歧**（各仓拉齐基线）：

1. **n06 签名带 `=`**：公开结构知识 → `protocol`（明确），非验签模糊
2. **n13 DEK 载荷 key 长度错**：载荷结构在解包后才可见，除 alg 族不符（D8 明确）
   外一律 `decrypt-failed`（模糊，I7 保守默认）
3. **n10 digest 未入 signedHeaders（I1）**：结构前置校验，`protocol`（明确）

## 负样本与故障注入手册的映射

| 样本 | 手册（docs/fault-injection-playbook.md） |
|---|---|
| n01 | P1 信封密文损伤（digest+签名重算后直达 AEAD 层） |
| n12 | P2 信封截断/缺字段（结构层，与 n01 构成 I7 分界对照） |
| n13 | P3 DEK key 段畸形 |
| n11 | P4 响应声明跨族套件（声明与配置比对是公开结构知识，协议类明确拒绝） |
| n16 | P5 跨端点签名重放（`verifyPath` 覆盖） |
| n06 | P6 签名段 urlencode 污染 |
| p13(混合大小写) | P7 头名大小写混合 |

手册的网络层矩阵（N1–N6：拒连/超时/断连/5xx/头规范化/204）属传输适配器层，
不适合静态样本，各仓按手册用各自 mock 栈实现。

## 消费要求（每仓验收口径）

1. fixture 字节副本进仓（位置随语言惯例），CI 校验与真源 sha256 一致
2. build 方向：`byte-exact` 全量比对；`deterministic-fields` 按 opaque 剥离后比对
3. verify 方向：positive 断言明文一致；negative 断言错误分类（本仓错误码 →
   canonical class 的映射表须在测试中显式声明）
4. 条数哨兵 + 已知 id 哨兵（防 fixture 漂移静默通过）
5. 样本集升级（格式 v2 等）：新目录 `interop/v2/`，旧 SDK 对新样本
   "明确拒绝 + 协议版本类错误"本身是必测行为
