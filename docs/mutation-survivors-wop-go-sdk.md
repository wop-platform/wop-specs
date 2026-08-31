# wop-go-sdk 变异测试存活清单归档（2026-08-31）

> 来源：wop-go-sdk 质量闭环第四轮全量变异（自研 AST 引擎 `tools/mutation`，13 类算子）
> 基线：1350 变异点 = killed 910 / survived 265 / invalid 175（invalid 编译失败不计分母）
> 击杀率：口径A raw = 910/1175 = 77.4%；口径B（剔诊断文案存活）= 910/1072 = 84.9%
> CI：口径B 门禁 ≥80% 已入 `.github/workflows/mutation.yml`（PR 增量 + 每周全量）
> 注记：本清单基于第四轮副本；其后第四/五批补杀断言（keys 78/81、signheader 35/52、
> transport 41、verify 37/59、sm2Decrypt 253 系）已随仓提交，下次全量将转为 killed。

## 分组汇总

| 组 | 数量 | 论点 |
|---|---|---|
| diag | 103 | 诊断文案（newError/fuzzyError 参数，非对外契约，I7） |
| hexdead | 66 | hexUpperTable 死表项（保留字符走 WriteByte 短路，永不索引） |
| unreach | 11 | 不可达防御衍生（sealMessage/wrapDEK 在装配后调用链恒成功） |
| math | 12 | 数学兜底（验签等式 rCheck==r 使前置 r/s 范围检查语义冗余；t=0/x1=nil 后等式仍必然失败） |
| invariant | 38 | 哨兵/循环不变量（expired==0 前置转换、timestamp 0 哨兵、nonce 长度无契约、j≥1 恒真、CEK/IV 错误同码难区分） |
| prob | 3 | 概率等价（k=0 事件概率 2^-256；KDF 多算一块截断等值） |
| chain | 30 | 链式同码（空 map 时循环零次，等价空串） |
| review | 2 | 自指常量等价：测试用同一 maxResponseBytes 常量构造边界，常量变异后边界同移（11MiB 钉死值是否需向量级锚定 → spec 决策） |

合计 265（与 survived=265 对账）

## 逐条明细（组内按 文件:行 排序）

### diag —— 诊断文案（newError/fuzzyError 参数，非对外契约，I7）

| 算子 | 位置 | 变异 |
|---|---|---|
| LCR | client.go:63:36 | 字符串加后缀 mut |
| LCR | client.go:74:36 | 字符串加后缀 mut |
| LCR | client.go:150:47 | 字符串加后缀 mut |
| LCR | client.go:153:47 | 字符串加后缀 mut |
| LCR | client.go:178:48 | 字符串加后缀 mut |
| LCR | client.go:195:47 | 字符串加后缀 mut |
| LCR | client.go:231:15 | 字符串加后缀 mut |
| LCR | client.go:231:40 | 字符串加后缀 mut |
| LCR | client.go:235:15 | 字符串加后缀 mut |
| LCR | client.go:235:40 | 字符串加后缀 mut |
| LCR | digest.go:37:10 | 字符串加后缀 mut |
| LCR | digest.go:37:14 | 字符串加后缀 mut |
| LCR | digest.go:38:4 | 字符串加后缀 mut |
| LCR | digest.go:53:4 | 字符串加后缀 mut |
| LCR | digest.go:67:4 | 字符串加后缀 mut |
| LCR | digest.go:71:39 | 字符串加后缀 mut |
| LCR | encoding.go:23:38 | 字符串加后缀 mut |
| LCR | encoding.go:27:38 | 字符串加后缀 mut |
| LCR | keyencrypt.go:25:11 | 字符串加后缀 mut |
| LCR | keyencrypt.go:25:36 | 字符串加后缀 mut |
| LCR | keyencrypt.go:29:11 | 字符串加后缀 mut |
| LCR | keyencrypt.go:34:11 | 字符串加后缀 mut |
| LCR | keyencrypt.go:34:36 | 字符串加后缀 mut |
| LCR | keyencrypt.go:38:11 | 字符串加后缀 mut |
| LCR | keyencrypt.go:54:37 | 字符串加后缀 mut |
| LCR | keyencrypt.go:63:37 | 字符串加后缀 mut |
| LCR | keys.go:24:36 | 字符串加后缀 mut |
| LCR | keys.go:38:36 | 字符串加后缀 mut |
| LCR | keys.go:50:36 | 字符串加后缀 mut |
| LCR | keys.go:54:36 | 字符串加后缀 mut |
| LCR | keys.go:66:36 | 字符串加后缀 mut |
| LCR | keys.go:70:36 | 字符串加后缀 mut |
| LCR | keys.go:82:36 | 字符串加后缀 mut |
| LCR | keys.go:86:36 | 字符串加后缀 mut |
| LCR | keys.go:98:36 | 字符串加后缀 mut |
| LCR | keys.go:103:36 | 字符串加后缀 mut |
| LCR | keys.go:115:31 | 字符串加后缀 mut |
| LCR | message.go:25:36 | 字符串加后缀 mut |
| LCR | message.go:37:36 | 字符串加后缀 mut |
| LCR | message.go:49:36 | 字符串加后缀 mut |
| LCR | message.go:59:36 | 字符串加后缀 mut |
| LCR | message.go:63:36 | 字符串加后缀 mut |
| LCR | message.go:91:47 | 字符串加后缀 mut |
| LCR | message.go:95:47 | 字符串加后缀 mut |
| LCR | message.go:99:47 | 字符串加后缀 mut |
| LCR | message.go:103:47 | 字符串加后缀 mut |
| LCR | message.go:106:47 | 字符串加后缀 mut |
| LCR | message.go:109:47 | 字符串加后缀 mut |
| LCR | signature.go:36:11 | 字符串加后缀 mut |
| LCR | signature.go:36:36 | 字符串加后缀 mut |
| LCR | signature.go:40:11 | 字符串加后缀 mut |
| LCR | signature.go:45:11 | 字符串加后缀 mut |
| LCR | signature.go:45:36 | 字符串加后缀 mut |
| LCR | signature.go:50:11 | 字符串加后缀 mut |
| LCR | signature.go:64:33 | 字符串加后缀 mut |
| LCR | signature.go:70:32 | 字符串加后缀 mut |
| LCR | signature.go:78:32 | 字符串加后缀 mut |
| LCR | signheader.go:32:47 | 字符串加后缀 mut |
| LCR | signheader.go:36:47 | 字符串加后缀 mut |
| LCR | signheader.go:43:4 | 字符串加后缀 mut |
| LCR | signheader.go:46:47 | 字符串加后缀 mut |
| LCR | signheader.go:50:47 | 字符串加后缀 mut |
| LCR | signheader.go:53:47 | 字符串加后缀 mut |
| LCR | signheader.go:58:47 | 字符串加后缀 mut |
| LCR | signheader.go:61:47 | 字符串加后缀 mut |
| LCR | signheader.go:103:10 | 字符串加后缀 mut |
| LCR | signheader.go:103:14 | 字符串加后缀 mut |
| LCR | signheader.go:103:41 | 字符串加后缀 mut |
| LCR | signheader.go:107:10 | 字符串加后缀 mut |
| LCR | signheader.go:107:14 | 字符串加后缀 mut |
| LCR | signheader.go:107:41 | 字符串加后缀 mut |
| LCR | signheader.go:140:10 | 字符串加后缀 mut |
| LCR | signheader.go:140:37 | 字符串加后缀 mut |
| LCR | signheader.go:143:10 | 字符串加后缀 mut |
| LCR | signheader.go:143:37 | 字符串加后缀 mut |
| LCR | sm2raw.go:95:27 | 字符串加后缀 mut |
| LCR | sm2raw.go:100:26 | 字符串加后缀 mut |
| LCR | sm2raw.go:111:25 | 字符串加后缀 mut |
| LCR | sm2raw.go:196:27 | 字符串加后缀 mut |
| LCR | sm2raw.go:201:26 | 字符串加后缀 mut |
| LCR | sm2raw.go:212:25 | 字符串加后缀 mut |
| LCR | sm2raw.go:254:26 | 字符串加后缀 mut |
| LCR | sm2raw.go:257:26 | 字符串加后缀 mut |
| LCR | sm2raw.go:262:26 | 字符串加后缀 mut |
| LCR | sm2raw.go:278:26 | 字符串加后缀 mut |
| LCR | sm2raw.go:289:26 | 字符串加后缀 mut |
| LCR | suite.go:60:44 | 字符串加后缀 mut |
| LCR | suite.go:65:4 | 字符串加后缀 mut |
| LCR | suite.go:69:50 | 字符串加后缀 mut |
| LCR | suite.go:72:50 | 字符串加后缀 mut |
| LCR | suite.go:78:4 | 字符串加后缀 mut |
| LCR | transport.go:44:52 | 字符串加后缀 mut |
| LCR | transport.go:49:52 | 字符串加后缀 mut |
| LCR | transport.go:60:52 | 字符串加后缀 mut |
| LCR | transport.go:66:52 | 字符串加后缀 mut |
| LCR | transport.go:69:54 | 字符串加后缀 mut |
| LCR | verify.go:38:44 | 字符串加后缀 mut |
| LCR | verify.go:53:4 | 字符串加后缀 mut |
| LCR | verify.go:64:51 | 字符串加后缀 mut |
| LCR | verify.go:67:45 | 字符串加后缀 mut |
| LCR | verify.go:70:44 | 字符串加后缀 mut |
| LCR | verify.go:78:45 | 字符串加后缀 mut |
| LCR | verify.go:115:4 | 字符串加后缀 mut |

### hexdead —— hexUpperTable 死表项（保留字符走 WriteByte 短路，永不索引）

| 算子 | 位置 | 变异 |
|---|---|---|
| LCR | encoding.go:89:62 | 字符串加后缀 mut |
| LCR | encoding.go:89:80 | 字符串加后缀 mut |
| LCR | encoding.go:89:86 | 字符串加后缀 mut |
| LCR | encoding.go:90:2 | 字符串加后缀 mut |
| LCR | encoding.go:90:8 | 字符串加后缀 mut |
| LCR | encoding.go:90:14 | 字符串加后缀 mut |
| LCR | encoding.go:90:20 | 字符串加后缀 mut |
| LCR | encoding.go:90:26 | 字符串加后缀 mut |
| LCR | encoding.go:90:32 | 字符串加后缀 mut |
| LCR | encoding.go:90:38 | 字符串加后缀 mut |
| LCR | encoding.go:90:44 | 字符串加后缀 mut |
| LCR | encoding.go:90:50 | 字符串加后缀 mut |
| LCR | encoding.go:90:56 | 字符串加后缀 mut |
| LCR | encoding.go:91:8 | 字符串加后缀 mut |
| LCR | encoding.go:91:14 | 字符串加后缀 mut |
| LCR | encoding.go:91:20 | 字符串加后缀 mut |
| LCR | encoding.go:91:26 | 字符串加后缀 mut |
| LCR | encoding.go:91:32 | 字符串加后缀 mut |
| LCR | encoding.go:91:38 | 字符串加后缀 mut |
| LCR | encoding.go:91:44 | 字符串加后缀 mut |
| LCR | encoding.go:91:50 | 字符串加后缀 mut |
| LCR | encoding.go:91:56 | 字符串加后缀 mut |
| LCR | encoding.go:91:62 | 字符串加后缀 mut |
| LCR | encoding.go:91:68 | 字符串加后缀 mut |
| LCR | encoding.go:91:74 | 字符串加后缀 mut |
| LCR | encoding.go:91:80 | 字符串加后缀 mut |
| LCR | encoding.go:91:86 | 字符串加后缀 mut |
| LCR | encoding.go:91:92 | 字符串加后缀 mut |
| LCR | encoding.go:92:2 | 字符串加后缀 mut |
| LCR | encoding.go:92:8 | 字符串加后缀 mut |
| LCR | encoding.go:92:14 | 字符串加后缀 mut |
| LCR | encoding.go:92:20 | 字符串加后缀 mut |
| LCR | encoding.go:92:26 | 字符串加后缀 mut |
| LCR | encoding.go:92:32 | 字符串加后缀 mut |
| LCR | encoding.go:92:38 | 字符串加后缀 mut |
| LCR | encoding.go:92:44 | 字符串加后缀 mut |
| LCR | encoding.go:92:50 | 字符串加后缀 mut |
| LCR | encoding.go:92:56 | 字符串加后缀 mut |
| LCR | encoding.go:92:62 | 字符串加后缀 mut |
| LCR | encoding.go:92:92 | 字符串加后缀 mut |
| LCR | encoding.go:93:8 | 字符串加后缀 mut |
| LCR | encoding.go:93:14 | 字符串加后缀 mut |
| LCR | encoding.go:93:20 | 字符串加后缀 mut |
| LCR | encoding.go:93:26 | 字符串加后缀 mut |
| LCR | encoding.go:93:32 | 字符串加后缀 mut |
| LCR | encoding.go:93:38 | 字符串加后缀 mut |
| LCR | encoding.go:93:44 | 字符串加后缀 mut |
| LCR | encoding.go:93:50 | 字符串加后缀 mut |
| LCR | encoding.go:93:56 | 字符串加后缀 mut |
| LCR | encoding.go:93:62 | 字符串加后缀 mut |
| LCR | encoding.go:93:68 | 字符串加后缀 mut |
| LCR | encoding.go:93:74 | 字符串加后缀 mut |
| LCR | encoding.go:93:80 | 字符串加后缀 mut |
| LCR | encoding.go:93:86 | 字符串加后缀 mut |
| LCR | encoding.go:93:92 | 字符串加后缀 mut |
| LCR | encoding.go:94:2 | 字符串加后缀 mut |
| LCR | encoding.go:94:8 | 字符串加后缀 mut |
| LCR | encoding.go:94:14 | 字符串加后缀 mut |
| LCR | encoding.go:94:20 | 字符串加后缀 mut |
| LCR | encoding.go:94:26 | 字符串加后缀 mut |
| LCR | encoding.go:94:32 | 字符串加后缀 mut |
| LCR | encoding.go:94:38 | 字符串加后缀 mut |
| LCR | encoding.go:94:44 | 字符串加后缀 mut |
| LCR | encoding.go:94:50 | 字符串加后缀 mut |
| LCR | encoding.go:94:56 | 字符串加后缀 mut |
| LCR | encoding.go:94:62 | 字符串加后缀 mut |

### unreach —— 不可达防御衍生（sealMessage/wrapDEK 在装配后调用链恒成功）

| 算子 | 位置 | 变异 |
|---|---|---|
| COI | client.go:239:2 | if cond → if false |
| LCR | client.go:240:15 | 字符串加后缀 mut |
| COI | client.go:245:2 | if cond → if false |
| LCR | client.go:246:15 | 字符串加后缀 mut |
| COI | message.go:58:2 | if cond → if false |
| COI | message.go:62:2 | if cond → if false |
| ROI | sm2raw.go:161:10 | false → true |
| OKN | sm2raw.go:228:27 | 0 → 1 |
| ROI | sm2raw.go:229:15 | false → true |
| OKN | sm2raw.go:277:26 | 0 → 1 |
| LCR | verify.go:23:59 | 字符串加后缀 mut |

### math —— 数学兜底（验签等式 rCheck==r 使前置 r/s 范围检查语义冗余；t=0/x1=nil 后等式仍必然失败）

| 算子 | 位置 | 变异 |
|---|---|---|
| COI | sm2raw.go:147:2 | if cond → if false |
| OBB | sm2raw.go:147:14 | <= → < |
| LOR | sm2raw.go:147:19 | || → && |
| OBB | sm2raw.go:147:31 | <= → < |
| LOR | sm2raw.go:147:36 | || → && |
| OBB | sm2raw.go:147:48 | >= → > |
| OKN | sm2raw.go:147:51 | 0 → 1 |
| LOR | sm2raw.go:147:53 | || → && |
| OBB | sm2raw.go:147:65 | >= → > |
| OKN | sm2raw.go:147:68 | 0 → 1 |
| COI | sm2raw.go:152:2 | if cond → if false |
| COI | sm2raw.go:160:2 | if cond → if false |

### invariant —— 哨兵/循环不变量（expired==0 前置转换、timestamp 0 哨兵、nonce 长度无契约、j≥1 恒真、CEK/IV 错误同码难区分）

| 算子 | 位置 | 变异 |
|---|---|---|
| OBB | client.go:73:13 | < → <= |
| OKN | client.go:73:15 | 0 → 1 |
| COI | client.go:165:2 | if cond → if false |
| OKN | client.go:165:18 | 0 → 1 |
| OBB | client.go:254:29 | < → <= |
| SDEL | encoding.go:50:2 | 删除表达式语句 |
| ROI | encoding.go:51:15 | false → true |
| SDEL | encoding.go:72:2 | 删除表达式语句 |
| OKN | encoding.go:86:22 | 256 → 257 |
| OKN | keys.go:32:12 | 1 → 2 |
| OKN | sm2raw.go:46:13 | 0 → 1 |
| SBR | sm2raw.go:46:29 | >> → << |
| OKN | sm2raw.go:46:32 | 8 → 9 |
| OKN | sm2raw.go:85:22 | 8 → 9 |
| OKN | sm2raw.go:102:11 | 0 → 1 |
| OKN | sm2raw.go:176:11 | 0 → 1 |
| SBR | sm2raw.go:176:24 | >> → << |
| OKN | sm2raw.go:176:27 | 24 → 25 |
| OKN | sm2raw.go:177:11 | 1 → 2 |
| SBR | sm2raw.go:177:24 | >> → << |
| OKN | sm2raw.go:177:27 | 16 → 17 |
| OKN | sm2raw.go:178:11 | 2 → 3 |
| SBR | sm2raw.go:178:24 | >> → << |
| OKN | sm2raw.go:178:27 | 8 → 9 |
| OKN | sm2raw.go:188:25 | 8 → 9 |
| OKN | sm2raw.go:203:11 | 0 → 1 |
| ROI | sm2raw.go:221:13 | true → false |
| OKN | sm2raw.go:223:11 | 0 → 1 |
| BRK | sm2raw.go:225:4 | break → continue |
| OKN | sm2raw.go:241:25 | 65 → 66 |
| AOR | sm2raw.go:241:27 | + → - |
| OKN | sm2raw.go:241:28 | 32 → 33 |
| AOR | sm2raw.go:241:30 | + → - |
| ROI | sm2raw.go:270:13 | true → false |
| OKN | sm2raw.go:272:11 | 0 → 1 |
| BRK | sm2raw.go:274:4 | break → continue |
| OKN | sm2raw.go:295:23 | 64 → 65 |
| OKN | transport.go:64:69 | 1 → 2 |

### prob —— 概率等价（k=0 事件概率 2^-256；KDF 多算一块截断等值）

| 算子 | 位置 | 变异 |
|---|---|---|
| COI | sm2raw.go:78:3 | if cond → if true |
| OBB | sm2raw.go:78:15 | > → >= |
| OBB | sm2raw.go:172:15 | < → <= |

### chain —— 链式同码（空 map 时循环零次，等价空串）

| 算子 | 位置 | 变异 |
|---|---|---|
| COI | canonical.go:11:2 | if cond → if false |
| COI | digest.go:48:2 | if cond → if false |
| COI | digest.go:62:2 | if cond → if false |
| COI | encoding.go:22:2 | if cond → if false |
| COI | keys.go:37:2 | if cond → if false |
| COI | keys.go:45:2 | if cond → if false |
| COI | keys.go:49:2 | if cond → if false |
| COI | keys.go:61:2 | if cond → if false |
| COI | keys.go:65:2 | if cond → if false |
| COI | keys.go:78:2 | if cond → if false |
| COI | keys.go:81:2 | if cond → if false |
| LOR | keys.go:81:20 | || → && |
| OKN | message.go:95:118 | 0 → 1 |
| COI | message.go:98:2 | if cond → if false |
| COI | message.go:102:2 | if cond → if false |
| OKN | message.go:106:106 | 0 → 1 |
| COI | signature.go:60:2 | if cond → if false |
| COI | signheader.go:31:2 | if cond → if false |
| LCR | signheader.go:31:16 | 字符串加后缀 mut |
| OBB | signheader.go:35:8 | <= → < |
| OKN | signheader.go:35:11 | 0 → 1 |
| OKN | signheader.go:46:88 | 0 → 1 |
| OKN | signheader.go:50:79 | 1 → 2 |
| OBB | signheader.go:52:43 | > → >= |
| LCR | transport.go:41:41 | 字符串加后缀 mut |
| LCR | verify.go:37:29 | 字符串加后缀 mut |
| COI | verify.go:48:2 | if cond → if false |
| OKN | verify.go:59:29 | 0 → 1 |
| COI | verify.go:104:2 | if cond → if false |
| COI | verify.go:122:2 | if cond → if false |

### review —— 自指常量等价：测试用同一 maxResponseBytes 常量构造边界，常量变异后边界同移（11MiB 钉死值是否需向量级锚定 → spec 决策）

| 算子 | 位置 | 变异 |
|---|---|---|
| OKN | transport.go:33:26 | 11 → 12 |
| OKN | transport.go:33:32 | 20 → 21 |

## 提请评审的决策点

1. **transport.go:33（review 组）**：11MiB 限额常量与测试自指，变异不可见。若需钉死具体数值，
   应在 crypto-vectors 或 spec 附录补字面量锚（现状：数值变更仅被 CI 覆盖率/评审捕捉）。
2. **sm2raw.go:147（math 组）**：sm2Verify 前置 r/s 范围检查与验证等式数学冗余（fail-fast 优化）。
   建议：保留（防御深度+快速失败），spec 附录 D 记录其「性能优化非安全边界」定位。
3. **sm2raw.go:253（对照参考）**：解密入口长度前置 65+32 不可省——跳过即切片越界 panic。
   建议：spec 附录 D 增补「解密入口长度前置为必要边界」条款，跨语言 SDK 横评必查。
4. **diag 组（103 条）**：错误文案在 I7 纪律下非契约，口径B 已从分母剔除。
   建议：spec 明文「错误文案非对外契约，禁止测试断言完整文案（关键词锚点除外）」。