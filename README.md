# RollingMinHash

滚动哈希 + 分块 MinHash 加权平均签名工具。为大批量中文/中英混合文本文档（如书籍 markdown、txt）计算确定性、可复现的 128 维签名向量，用于近似查重 / 相似度比较。

## 算法流程

严格按顺序执行：

1. **分块清理**：原文按 `--chunk-size` 字（原文字符数，默认 2000）分块，逐块清理——剔标点/空格/换行，汉字（含扩展区）保留、英文字母转小写，得到纯文字的 Unicode scalar 值流。
2. **滚动哈希滑窗**：每块用 Rabin-Karp 式多项式滚动哈希扫描，窗口哈希 `h = (h·B − v_out·B^k + v_in) mod 2^64` O(1) 更新（基数 `B = 0x100000001B3`）。每个窗口哈希放入 `Set<UInt64>` 自动去重，得到字符级 k-shingle 集合（默认 `k=3`）。
3. **停用词过滤**：停用词表用同一滚动哈希公式预编码成哈希集合，O(1) 判定；全虚字窗口（窗口内 k 字都属虚字表）与英文高频功能 3-gram（`the`/`and`/`ing` 等）同样剔除。
4. **MinHash 签名**：对最终集合用 128 个独立仿射哈希函数 `h_i(x) = a_i·x + b_i (mod 2^64)` 取最小值，得到每块一个 128 维签名（归一化到 [0,1)）。
5. **加权平均**：以每块清理后的有效字数为权重，对所有分块签名逐维加权平均，得到 128 维全书平均向量，即文件的最终 MinHash 签名。

确定性由固定 `--seed`（splitmix64 生成仿射哈希参数族）保证：同参数下同文件永远得到相同向量。

## 编译

依赖同仓库的 `CoreWordKit.swift`（停用词表 / 虚字边界字符表）：

```bash
swiftc -O -o /tmp/bookminhash CoreWordKit.swift RollingMinHash.swift
```

## 使用

```bash
# 处理单个文件或目录（目录递归扫描，跳过隐藏目录，默认只处理 .md/.txt）
bookminhash /path/to/books

# 写 JSON 报告（含每文件 128 维平均向量）
bookminhash /path/to/books --report /tmp/rmh-report.json

# 自定义参数
bookminhash /path/to/books --chunk-size 3000 --shingle-k 4 --dim 256 \
  --stopwords my_stops.txt --ext md,txt,html

# 运行自检
bookminhash --self-test
```

## 选项

| 选项 | 说明 | 默认 |
|---|---|---|
| `--chunk-size N` | 分块长度（原文字符数） | 2000 |
| `--shingle-k N` | 滑窗长度 | 3 |
| `--dim N` | MinHash 签名维度 | 128 |
| `--seed N` | 哈希族随机种子（固定则可复现） | `0x5EED_C0DE_1234_5678` |
| `--no-stopwords` | 关闭停用词/全虚字过滤 | 开启 |
| `--stopwords PATH` | 追加自定义停用词文件 | — |
| `--signatures` | 报告中附带每个分块的 128 维签名 | 不附带 |
| `--report PATH` | 写 JSON 报告 | — |
| `--ext LIST` | 目录扫描扩展名（逗号分隔） | `md,txt` |
| `--limit N` | 最多处理 N 个文件 | 不限 |
| `--self-test` | 运行自检 | — |

## 输出

- **stdout**：每文件一行 `路径  v=[前3维,…]  blocks=有效块/总块  dim=128`；进度日志走 stderr。
- **JSON 报告**（`--report`）：包含全局参数（chunkSize/shingleK/dim/seed/rollingBase/useStopwords）与每文件结果（字节数、字符数、块数、128 维平均向量 `vector`、各块明细 `blocks`）。加 `--signatures` 时各块附完整签名。
- 空集合块返回全 1 哨兵签名且不参与加权；无任何有效块的文件返回全 0 向量。

## 自检覆盖

`--self-test` 验证：文本清理规则、按原文字符数分块、滚动哈希 O(1) 增量公式与逐窗口直接计算的一致性（k=2~5）、shingle 集合去重、停用词哈希预编码与全虚字过滤、签名确定性/空集合哨兵、加权平均公式、端到端真实文件分块与确定性。

## 已知限制

- 仅处理 UTF-8 文本，非 UTF-8 文件被跳过并记录到 stderr。
- 分块按原文字符数切分，块边界不考虑段落/句子语义。
- 中文为主设计：全角字母/数字被清理剔除。
