# WOP Specs
![CodeRabbit Pull Request Reviews](https://img.shields.io/coderabbit/prs/github/wop-platform/wop-specs?utm_source=oss&utm_medium=github&utm_campaign=wop-platform%2Fwop-specs&labelColor=171717&color=FF570A&link=https%3A%2F%2Fcoderabbit.ai&label=CodeRabbit+Reviews)

万联易达开放平台（WOP）**对外公开规格文档集**：协议契约、SDK 统一规格与跨语言测试向量的唯一真源。商户与第三方开发者据此集成。

## 📚 文档目录

| 文档 | 版本 | 状态 | 说明 |
|------|------|------|------|
| [crypto/crypto-strategy-spec.md](crypto/crypto-strategy-spec.md) | v0.4-draft | 已评审冻结（D1–D15，2026-09-01） | 加密协议契约：`securityReq` 算法套件、四维算法策略、线上字节格式、密钥分发编码、协议不变式 I1–I7、错误分类 |
| [sdk/wop-sdk-spec.md](sdk/wop-sdk-spec.md) | v1.0-ratified | 已批准 | 各语言官方 SDK 统一规格：功能面 F1–F9、概念 API、§2.1 出向必传 header 契约、§2.2 WopError 七值闭集、每语言密码依赖白名单、附录 D 跨语言勘误纪律（D1–D7）、附录 E 质量任务契约（E1–E3）、验收标准 A1–A7 |
| [crypto/crypto-vectors.json](crypto/crypto-vectors.json) | 2026-08-28 | 稳定 | 黄金测试向量（TEST-ONLY 密钥）：跨语言**字节级**断言基准，防实现漂移的验收载体（D9） |
| [interop/v1/](interop/v1/) | wop-interop-1 | 冻结 | 协议编排互操作样本集（30 条：6 build + 7 positive + 17 negative）：canonicalRequest、signedHeaders、L2 信封与 canonical 错误分类的跨仓一致性合同 |
| [docs/fault-injection-playbook.md](docs/fault-injection-playbook.md) | 1.0 | 稳定 | 故障注入测试手册：协议层 P1–P7 + 网络层 N1–N6 注入矩阵，I7 明确/模糊分界的测试锚 |
| [docs/mutation-survivors-wop-go-sdk.md](docs/mutation-survivors-wop-go-sdk.md) | 2026-08-31 | 归档 | wop-go-sdk 变异测试存活清单：265 幸存体逐条等价性论证（diag/hexdead/unreach/math/invariant/prob/chain/review 八组） |

**阅读顺序**：加密协议 spec（协议真源）→ SDK 规格（协议的客户端封装契约）→ 黄金向量（正确性锚）→ interop 样本集（编排一致性锚）。各文档版本互相引用，升级须同步（联动清单见「规格治理」第 4 条）。

## 🛠️ 官方 SDK 实现矩阵

协议与 SDK 规格的参考实现，全部通过黄金向量字节级合规 + 负向量拒绝矩阵 + 覆盖率门禁（≥98%）：

| 语言 | 仓库 | 套件支持 | 覆盖率（行/分支） |
|------|------|----------|-------------------|
| Java | [wop-java-sdk](https://github.com/wop-platform/wop-java-sdk) | RSA3072/4096 + SM2-SM3 | 100% / 98.4% |
| Go | [wop-go-sdk](https://github.com/wop-platform/wop-go-sdk) | RSA3072/4096 + SM2-SM3 | 98.6%（语句）+ 显式分支矩阵 |
| TypeScript | [wop-typescript-sdk](https://github.com/wop-platform/wop-typescript-sdk) | RSA3072/4096（SM2-SM3 列路线图） | 100% / 100% |
| Python | [wop-python-sdk](https://github.com/wop-platform/wop-python-sdk) | RSA3072/4096 + SM2-SM3 | 100% / 100% |
| PHP | [wop-php-sdk](https://github.com/wop-platform/wop-php-sdk) | RSA3072/4096（SM2-SM3 列路线图） | 门禁达标（PHPUnit） |
| .NET | [wop-dotnet-sdk](https://github.com/wop-platform/wop-dotnet-sdk) | RSA3072/4096 + SM2-SM3 | 99.4% / 99.1% |

> TypeScript / PHP 首版仅 RSA 套件为国密生态库成熟度所限（spec 裁决 Q7）：SM2/SM3/SM4-GCM 在两语言生态无主流库支持，官方 SDK 将以纯实现补齐（D11：官方 SDK 即 SM 生态答案），以黄金向量为唯一正确性锚。

## 🔐 协议一页速览

- **套件声明**：`x-wop-sign` 中的 `securityReq` 三段式 `WOP-<密钥族>-<摘要族>`，合法组合仅 `WOP-RSA3072-SHA256`、`WOP-RSA4096-SHA256`、`WOP-SM2-SM3`，跨族拒绝
- **四维算法**：签名 / 报文加密（L2 数字信封 bulk）/ DEK 包装 / 内容摘要，由套件一次性原子推导
- **线上编码**：Base64URL 无填充（拒收 `=`）；hex 一律小写；SM2 签名裸 r‖s 64B、密文 C1C3C2，线上禁 ASN.1/DER
- **安全时序**：先验签后解密；DEK alg 族比对先于 bulk 解密；digest 必入签名头（I1）
- **报文安全等级**：L0 明文 + 摘要 / L2 全文数字信封（AES-256-GCM 或 SM4-GCM）
- **错误纪律**：公开协议知识报错明确，密钥参与判定的失败一律模糊（防 oracle，I7）

## 🔢 条款编号索引

多系列编号在两份 spec 内是**独立空间**（D / E / Q 三组存在同号不同义），跨文档引用必须带文档前缀（如「crypto-spec D14」「sdk-spec D6」），避免同号歧义：

| 前缀 | 系列 | 定义处 |
|------|------|--------|
| crypto-spec R1–R5 | 需求条目 | crypto-strategy-spec §1.2 |
| crypto-spec D1–D15 | 协议评审决策 | crypto-strategy-spec 附录 C |
| crypto-spec E1–E5 | 扩展性需求 | crypto-strategy-spec §8 |
| crypto-spec I1–I7 | 协议不变式 | crypto-strategy-spec §10.1 |
| crypto-spec Q1–Q5 | 已关闭五问 | crypto-strategy-spec §9（决策已并入附录 C） |
| sdk-spec Q1–Q7 | SDK 裁决记录 | wop-sdk-spec 文件头 + §6 |
| sdk-spec F1–F9 | SDK 功能面 | wop-sdk-spec §1.3 |
| sdk-spec D1–D7 | 跨语言实现勘误纪律 | wop-sdk-spec 附录 D |
| sdk-spec E1–E3 | 跨语言质量任务契约 | wop-sdk-spec 附录 E |
| sdk-spec G1–G3 | canonicalRequest 拼装规则 | wop-sdk-spec 附录 G |
| sdk-spec A1–A7 | 每仓验收标准 | wop-sdk-spec §5 |
| playbook P1–P7 / N1–N6 | 故障注入场景 | fault-injection-playbook §1/§2 |

## 📐 版本与变更策略

- spec 采用**冻结版本 + 决策记录**制：每条关键裁决有唯一编号（协议 D1–D15、SDK Q1–Q7），正文与决策记录同步演进，不悄悄改
- 黄金向量变更 = 破坏性变更：须 bump 向量版本并同步六个 SDK 仓 fixture，CI 全红即拦截漂移
- 本仓库不含任何内部实现细节；网关实现侧文档不在公开范围

## 📄 License

[MIT](LICENSE)

## 规格治理（2026-08-29 起）

1. **单一来源**：`sdk/wop-sdk-spec.md`、`crypto/crypto-strategy-spec.md`、`crypto/crypto-vectors.json`
   以本仓（wop-specs）为唯一维护版；各实现仓（网关 / 六仓 SDK）内的同名文件是同步副本，
   副本出现分歧时以本仓为准。
2. **向量变更走 PR**：修改 `crypto/crypto-vectors.json` 的 commit **必须通过 PR 合并进本仓后**，
   各实现仓才能做对应的代码/测试变更（先合 PR、后改码）。
3. **副本同步**：各仓 CI 必须含"本仓真源 vs 本地副本"字节比对，不一致即 fail（禁止降级为 warning）。
4. **spec 版本事件联动清单**：任何决策钉死 / 版本 bump / 状态变更合入后，必须逐项核对下列索引面并同 PR（或紧随 PR）刷新（全仓 `grep 'v0\.'` 自查）：
   - 本 README 文档目录表的「版本 / 状态」列；
   - `docs/fault-injection-playbook.md` 头注「协议依据」；
   - `interop/v1/interop-cases.json` 的 `_meta.specVersion`——**例外：禁手改、不随 spec 事件直接刷新**，仅随样本集六仓协同**再生成**刷新（单仓改字节会分叉冻结合同）；spec 版本事件时只核对 interop/v1/README.md 现状注记是否仍成立（样本字节语义是否受影响），语义未变则维持冻结；
   - 各实现仓 README 及其他引用旧版本号的文档。
   - 决策编号范围引用面（spec 头注状态行与本 README 三处索引面）——按第 5 条 grep 自查归零。

5. **决策范围引用联动核查**：决策编号范围上界变更（如 D14→D15、Q6→Q7）的 PR，合并前必须以旧上界
   为模式（D15 例：`grep -rn 'D1[–-]D14' .`，全角 `–` 与半角 `-` 都查）全仓自查，旧范围引用归零方可合并。
   已知引用面：spec 头注状态行、本 README 文档目录「状态」列、条款编号索引表、版本与变更策略节。
   教训来源：D15 留档（PR #10）仅更新索引表，漏目录与策略两处致 README 内部矛盾，由评审评论捕获。

6. **interop 样本集增删联动核查**：样本集增删（`_meta.caseCount` 变更）的 PR，合并前必须以旧计数为模式
   （30 条例：`grep -rn '29 条' .`）全仓自查，旧引用归零方可合并。引用面纪律：样本条数唯一硬编码面为
   本 README 文档目录（索引进）；spec 正文指针式引用——条数与清单以 `interop/v1/interop-cases.json`
   的 `_meta` 为唯一真源，不镜像样本 id 枚举（2026-09-02 指针化落地）。
   教训来源：n17 入集（PR #12）漏三处顶层引用（根 README 计数、sdk-spec §G3 条数、crypto-spec §10.3
   枚举），评审评论兜底而非规则拦截——与规则 5 教训同构（同类第二次）。
