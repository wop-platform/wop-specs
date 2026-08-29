# WOP Specs

万联易达开放平台（WOP）**对外公开规格文档集**：协议契约、SDK 统一规格与跨语言测试向量的唯一真源。商户与第三方开发者据此集成。

## 📚 文档目录

| 文档 | 版本 | 状态 | 说明 |
|------|------|------|------|
| [crypto/crypto-strategy-spec.md](crypto/crypto-strategy-spec.md) | v0.3-reviewed | 评审冻结（D1–D13） | 加密协议契约：`securityReq` 算法套件、四维算法策略、线上字节格式、密钥分发编码、协议不变式 I1–I7、错误分类 |
| [sdk/wop-sdk-spec.md](sdk/wop-sdk-spec.md) | v1.0-ratified | 已批准 | 各语言官方 SDK 统一规格：功能面 F1–F9、概念 API、每语言密码依赖白名单、验收标准 A1–A7 |
| [crypto/crypto-vectors.json](crypto/crypto-vectors.json) | 2026-08-28 | 稳定 | 黄金测试向量（TEST-ONLY 密钥）：跨语言**字节级**断言基准，防实现漂移的验收载体（D9） |

**阅读顺序**：加密协议 spec（协议真源）→ SDK 规格（协议的客户端封装契约）→ 黄金向量（正确性锚）。三份文档版本互相引用，升级须同步。

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

## 📐 版本与变更策略

- spec 采用**冻结版本 + 决策记录**制：每条关键裁决有唯一编号（协议 D1–D13、SDK Q1–Q7），正文与决策记录同步演进，不悄悄改
- 黄金向量变更 = 破坏性变更：须 bump 向量版本并同步六个 SDK 仓 fixture，CI 全红即拦截漂移
- 本仓库不含任何内部实现细节；网关实现侧文档不在公开范围

## 📄 License

[MIT](LICENSE)
