---
title: "RAG 数据层实战：用 MinerU 解析 PDF 年报"
date: 2026-06-09
tags: [RAG, PDF解析, MinerU, Python, TypeScript, 财报问答]
description: 以财报 RAG 为场景，剖析 PDF 解析为何是第一道坎——为何选 MinerU、如何用 HTTP 在 Python 与 TypeScript 之间划出清晰语言边界、快档配置如何节省一个数量级的解析时间，以及真实表格数据里藏着哪些噪声。
---

# RAG 数据层实战：用 MinerU 解析 PDF 年报

## 摘要

做 RAG，很多人把复杂度归结到模型和检索阶段。实际上第一道坎在数据入口：把 PDF 年报解析成结构化、可检索、可溯源的文本块。

这篇文章以财报问答系统为场景，拆开 RAG 数据层的第一步——「解析」。核心问题有三个：为什么选 MinerU？怎么用 HTTP 把 Python 与 TypeScript 的差异封在服务边界后面？快档配置如何把算力花在刀刃上？

文中包含真实运行数据（茅台 2024 年报解析统计）和两个落地工程坑，读完可以直接用。

---

## 一、PDF 是 RAG 的第一道坎

财报问答系统的数据源几乎只有一种形态：PDF 年报。而 PDF 是 RAG 里最难处理的入口格式，原因有三：

**版面复杂**：年报采用多栏排版，混有页眉、页脚、目录和图表。文字的视觉阅读顺序经常和 PDF 内部的字符存储顺序不一致。

**表格密集**：财报内容大量以表格形式呈现。后文统计显示，茅台 2024 年报的 content_list.json 中，表格块（`table` 类型）占所有条目约 12%，但一旦被当成普通文本展平，行列关系全部丢失——"营业收入"和它对应的数字就无法关联。

**数字版 vs 扫描件**：数字版 PDF 文字可直接抽取；扫描件是图片，需要 OCR。两种形态的处理路径完全不同，混用会大幅拖慢解析速度。

RAG 系统后续所有环节——分块、向量嵌入、检索、溯源——都以「有结构的文本块」为前提。解析这一步的产出质量，直接决定整个系统的天花板。

对解析的最低要求只有两条：

- 每个文本块携带**页码**（答案溯源到具体页依赖它）
- 每个块标明**是否为表格**（为后续对表格做专项处理）

---

## 二、选型：MinerU + HTTP 语言边界

### 为什么选 MinerU

解析工具选 **MinerU**，理由直接：开源免费、对中文和表格的识别效果是同类工具里最好的之一，正好打在财报场景（中文、表格密集）的强项上。

MinerU 的 GitHub 仓库：[opendatalab/MinerU](https://github.com/opendatalab/MinerU)

### 语言边界怎么划

MinerU 是 Python 生态，而整个应用主程序是 TypeScript（基于 Mastra 框架）。把 Python 库硬塞进 TS 工程是个陷阱——依赖管理、运行时、类型系统全部要兼容，维护成本极高。

正确做法是划一条清晰的语言边界：**Python 只负责 PDF → 结构化文本，通过 HTTP 暴露一个 `POST /parse` 接口；TS 侧只消费归一化后的数据契约，不碰任何版面或表格解析逻辑。**

TS 侧的契约定义如下：

```typescript
// src/lib/parser.ts
// 解析微服务（MinerU）的归一化输出契约。
// TS 侧只依赖这个形状——底层换引擎或升级 MinerU 版本，这里不需要改动。
export interface ParsedBlock {
  text: string;
  isTable: boolean;
}

export interface ParsedPage {
  page: number;    // 1-based，对应 PDF 页码
  blocks: ParsedBlock[];
}
```

只要 Python 服务输出的 JSON 符合 `{ pages: ParsedPage[] }`，TS 一行不动。接口处用契约对齐，两侧各用自己最擅长的语言。

---

## 三、快档配置：把算力花在刀刃上

MinerU 默认使用 `hybrid` 自动引擎，对所有类型的 PDF 都能处理，但速度慢。

巨潮资讯网下载的年报全是**数字版 PDF**，文字可直接抽取，根本不需要重型引擎。因此在解析服务里设置了一套「快档」配置，通过环境变量控制：

```python
# services/parser/app.py
# 数字版年报用 txt 抽取，比默认 hybrid-auto-engine 快一个数量级。
# 如果遇到扫描件，改用：MINERU_BACKEND=pipeline MINERU_METHOD=ocr
MINERU_BACKEND = os.environ.get("MINERU_BACKEND", "pipeline")
MINERU_METHOD  = os.environ.get("MINERU_METHOD",  "txt")
MINERU_LANG    = os.environ.get("MINERU_LANG",    "ch")
MINERU_TABLE   = os.environ.get("MINERU_TABLE",   "true")   # 表格是财报核心，必须保留
MINERU_FORMULA = os.environ.get("MINERU_FORMULA", "false")  # 财报几乎无公式，关掉省时
```

四个参数背后的取舍逻辑：

| 参数 | 取值 | 原因 |
|------|------|------|
| `method` | `txt` | 数字版直接抽文字，比 OCR / hybrid **快一个数量级** |
| `lang` | `ch` | 中文年报场景 |
| `table` | `true` | **表格是财报核心信息载体，必须保留** |
| `formula` | `false` | 财报几乎无公式，关掉节省解析时间 |

最终映射到 MinerU CLI：

```bash
mineru -p <pdf_path> -o <output_dir> -b pipeline -m txt -l ch -t true -f false
```

产物是 `<stem>/.../<stem>_content_list.json`，这是后续归一化的原料。

---

## 四、归一化：content_list → 契约形状

MinerU 输出的 `content_list.json` 是个「什么都有」的清单，包含 text、table、header、page_number、image、equation、list 等多种类型。`normalize()` 函数把它收敛成 TS 侧期望的形状：

```python
# services/parser/app.py
def normalize(content_list_path: Path) -> dict:
    items = json.loads(content_list_path.read_text(encoding="utf-8"))
    pages: dict[int, list[dict]] = {}

    for it in items:
        # MinerU 的 page_idx 是 0-based，转为 1-based 使答案页码与 PDF 原文对齐
        page = int(it.get("page_idx", 0)) + 1
        t = it.get("type")

        if t == "table":
            text = it.get("table_body") or it.get("table_caption") or ""
            is_table = True
        elif t == "text":
            text = it.get("text", "")
            is_table = False
        else:
            # header / page_number / image / equation / list 对财报问答无检索价值，过滤掉
            continue

        if not text.strip():
            continue  # 空块不送入嵌入模型，避免浪费 token

        pages.setdefault(page, []).append({"text": text, "isTable": is_table})

    return {"pages": [{"page": p, "blocks": b} for p, b in sorted(pages.items())]}
```

四个关键决定：

1. **页码 +1**：MinerU 的 `page_idx` 是 0-based，转成 1-based 后，答案里的"第 8 页"才能对上 PDF 原文
2. **只保留两种类型**：`table` 取 `table_body`、标 `isTable=true`；`text` 标 `isTable=false`
3. **其余类型过滤**：`header` / `page_number` / `image` 等不含财报问答所需信息，直接丢弃
4. **空块跳过**：避免把空白块送进嵌入模型，白烧 token

### 真实数据：maotai_2024 解析统计

茅台 2024 年报共 **2360 个条目**，类型分布如下：

| type | 数量 | 处理结果 |
|------|------|----------|
| `text` | 1796 | 保留（`isTable=false`） |
| `table` | 275 | 保留（`isTable=true`） |
| `header` | 143 | **丢弃** |
| `page_number` | 143 | **丢弃** |
| `image` | 2 | **丢弃** |
| `list` | 1 | **丢弃** |

最终保留 `1796 + 275 = 2071` 个块。被丢弃的 289 个条目里，`header` 和 `page_number` 各占 143 个——页眉里的公司名称和每页的页码在向量库里只会稀释检索质量。

**解析的本质是取舍**：在数据入口挡住噪声，比在检索端做补偿要便宜得多。

---

## 五、真实表格有多脏

MinerU 把表格输出为 HTML `<table>` 标签，不是 Markdown 表格。以茅台 2024 年报第 8 页主要财务数据表为例（原样输出）：

```html
<table>
  <tr><td>科目</td><td>本期数</td><td>上年同期数</td><td>变动比例(%)</td></tr>
  <tr><td>营业收入</td><td>170, 899, 152,276. 34</td><td>147, 693, 604, 994. 14</td><td>15.71</td></tr>
  <tr><td>营业成本</td><td>13, 789, 482,367. 98</td>...</tr>
</table>
```

注意 `170, 899, 152,276. 34`——这个数字本应是 `170,899,152,276.34`（约 1708.99 亿元的营业收入），但千分位逗号和小数点周围被 PDF 解析引擎误判为字符间距，产生了杂散空格。

这不是解析 bug，而是数字版 PDF 抽取的固有噪声。金融 RAG 里到处都有这种脏数据。当前阶段先如实记录，不急着清洗——后续讲「结构化清洗」时，这个案例正好是现成的反面教材。**先承认脏，才谈得上治脏。**

---

## 六、两个工程坑

### 坑 1：undici 默认超时不够用

MinerU 在 CPU 上解析一整本年报需要数分钟。Node.js 的 `undici`（Node 18+ 内置 fetch 的底层实现）默认 headers 超时是 5 分钟，稍大的年报直接超时报错。

解法是给解析请求单独配一个关掉 headers/body 超时的 dispatcher：

```typescript
// src/lib/parser.ts
import { Agent } from "undici";

// 对解析端点关闭读写超时：解析时长不可预测，等多久都行。
// connectTimeout 保留——连不上服务要快速失败，不要和"解析慢"混淆。
const parseDispatcher = new Agent({
  headersTimeout: 0,   // 0 = 无限等待
  bodyTimeout: 0,
  connectTimeout: 10_000,
});
```

### 坑 2：别每次都重新解析

解析这么慢，如果每次执行数据摄入流程都从头解析，等待体验会很差。

解析服务做了内容寻址缓存，缓存键 = `{文件名 stem}_{mtime}`：

```python
# services/parser/app.py
# mtime 变化说明文件内容变了，需要重新解析；否则直接复用已有结果。
key = f"{pdf_path.stem}_{int(pdf_path.stat().st_mtime)}"
# 命中已有 content_list.json 则跳过解析
```

只要 PDF 未被修改（mtime 不变），第二次以后的调用秒回。**慢操作 + 内容寻址缓存**，是离线数据流程的标配组合。

---

## 七、全流程示意

```mermaid
flowchart LR
    PDF[PDF 年报] -->|"MinerU CLI\n-b pipeline -m txt -l ch"| CL["content_list.json\n2360 条目"]
    CL -->|"normalize()\npage_idx+1 / 只留 text·table"| PG["pages:[{page, blocks:[{text, isTable}]}]"]
    PG -->|"HTTP POST /parse\nParsedPage 契约"| TS[TS 侧消费]
    note["扫描件 → -m ocr"] -.-> PDF
```

---

## 小结

| 决策点 | 结论 |
|--------|------|
| 解析工具 | MinerU，开源、中文表格识别最强 |
| 语言边界 | Python 服务 HTTP 暴露，TS 消费固定契约 |
| 快档配置 | 数字版用 `txt`，快一个数量级；扫描件用 `ocr` |
| 过滤策略 | 只保留 `text` + `table`，在入口挡住噪声 |
| 脏数据 | 表格输出为 HTML，数字含杂散空格，当前先如实记录 |
| 工程坑 | 关掉 undici 超时；mtime 缓存跳过重复解析 |

结构化文本就位之后，下一步是分块——一个 HTML 表格块可能包含几百行，怎么切才不把语义切坏？这是下一篇要处理的问题。

---

## 延伸阅读

- [MinerU GitHub 仓库](https://github.com/opendatalab/MinerU) — 官方源码，含 `backend / method / lang / table / formula` 各参数说明
- [MinerU 使用文档](https://mineru.readthedocs.io/) — 详细参数配置与 API 参考
- [undici Agent 配置文档](https://undici.nodejs.org/#/docs/api/Agent) — `headersTimeout` / `bodyTimeout` / `connectTimeout` 参数含义

---

## 行动号召

如果你在做文档问答或财报 RAG，欢迎在评论区或 [Issue](https://github.com/sggmico/build-blog/issues) 里交流踩过的解析坑——表格被展平、扫描件抽不出字、数字有空格噪声，这些在金融 RAG 场景里都很典型，值得展开讨论。
