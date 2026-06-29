---
title: "RAG 评估实战: 用语义指标和数值门禁验证财报问答"
date: 2026-06-29
tags: [RAG, RAGAS, LLM-as-Judge, Rerank, TypeScript]
description: "用 6 道财报问答对比 dense, hybrid 和 rerank, 拆解拒答, 引用溯源, 数值归一化与业务口径校验, 建立可复跑的 RAG 质量评估闭环."
---

# RAG 评估实战: 用语义指标和数值门禁验证财报问答

## 摘要

RAG 系统最危险的状态, 不是明显答错, 而是答案看起来正确, 数字却换了单位或业务口径. 这种问题很难靠人工浏览几条结果发现, 更不能靠一次演示证明系统可靠.

本文用一个 6 题财报问答集对比 dense, hybrid 和 rerank 三种检索方案. 结果显示, hybrid 在这组样本上没有带来收益, rerank 则把数值准确性从 0.833 提升到 1.000. 样本规模很小, 这些数值不是通用 benchmark. 真正可复用的是评估方法: 用语义指标定位链路问题, 用确定性代码守住财务数字, 再用人工抽检判断业务口径.

## 先定义什么叫答对

财报问答至少有 4 种失败方式:

1. 答案包含上下文没有提供的事实.
2. 答案与问题不相关.
3. 检索结果没有把关键证据排到前面.
4. 数字看起来接近, 但单位, 数量级或业务口径不一致.

前三类问题可以用 LLM 评估器辅助判断. 第四类不能只交给模型. 财务数字需要确定性程序校验, 业务口径还需要人工抽检.

因此, 本文使用 4 把尺子:

| 指标 | 检查对象 | 主要风险 |
| --- | --- | --- |
| Faithfulness | 答案与上下文 | 模型编造 |
| Answer relevancy | 问题与答案 | 答非所问 |
| Context relevance | 问题与召回块 | 检索噪声 |
| Numeric accuracy | 答案与标准数值 | 单位和数量级错误 |

[Ragas 官方指标列表](https://docs.ragas.io/en/stable/concepts/metrics/available_metrics/)包含 Faithfulness, Response Relevancy 和 Context Precision 等指标. 指标名称相似, 输入要求却不同. 例如 [Context Precision](https://docs.ragas.io/en/stable/concepts/metrics/available_metrics/context_precision/)关注相关块是否排在无关块之前, 使用前必须确认当前实现是否需要 reference answer. 不要仅凭名称替换指标.

## 把拒答和引用写进生成契约

财报数字会随年份和口径变化. 如果上下文没有答案, 模型从参数记忆中补出一个数字, 即使碰巧正确, 也无法审计.

生成 prompt 应明确 3 条约束:

1. 只依据召回片段回答.
2. 每个结论附公司, 年份和页码.
3. 上下文不足时返回固定拒答文本.

```ts
// src/generate/answer.ts
export function buildPrompt(
  question: string,
  chunks: RetrievedChunk[],
): string {
  return [
    // 限制证据边界, 避免模型使用无法追溯的参数记忆.
    "你是严谨的财报分析助手. 只依据下面的上下文片段回答问题.",
    "要求:",
    "1) 答案末尾标注引用, 格式为 (公司·年份年报·P页码);",
    // 固定拒答文本, 让评估程序能稳定识别 no-answer 结果.
    "2) 若上下文不足, 回答 未在所给资料中检索到, 不得补写数字.",
    `# 上下文片段\n${formatContext(chunks)}`,
    `# 问题\n${question}`,
  ].join("\n");
}
```

拒答不是失败. 如果语料中没有目标公司, 或答案块没有进入生成模型的可见窗口, 拒答是比猜测更正确的行为. 因此 no-answer 题要单独统计拒答成功率, 不能直接混入普通问答的语义均分.

引用也不能由生成层临时拼接. `company`, `period` 和 `sourcePage` 必须从解析阶段开始贯穿入库, 检索和生成.

```ts
// src/generate/answer.ts
export function formatContext(chunks: RetrievedChunk[]): string {
  return chunks
    .map(
      (chunk, index) =>
        // 在进入模型前绑定证据和来源, 防止答案与页码错配.
        `片段${index + 1}|${cn(chunk.meta.company)}·${chunk.meta.period}年报·P${chunk.meta.sourcePage}\n${chunk.text}`,
    )
    .join("\n\n");
}
```

![信息检索与答案生成链路](./assets/03_流程图-信息检索与答案生成.png)

## 数字相同不等于字符串相同

贵州茅台 2023 年营业收入可以写成 `147,693,604,994.14 元`, 也可以按万元口径写成 `14,769,360.50 万元`. 两者换算后接近同一数值, 直接比较字符串却会得到错误结论. 原始数据可回查[贵州茅台 2023 年年度报告](https://www.moutai.com.cn/mtgf/attachDir/2024/05/2024050716531754477.pdf).

OCR 还可能把 `994.14` 切成 `994. 14`. 模型则可能把完整金额概括为 `1476.94 亿`. 数值比较前至少要完成:

1. 清理数字内部的 OCR 空格.
2. 去掉千分位逗号.
3. 把万亿, 亿和万统一换算为元.
4. 使用显式相对误差处理合理四舍五入.

```ts
const UNIT_TO_YUAN: Record<string, number> = {
  万亿: 1e12,
  亿: 1e8,
  万: 1e4,
};

export function extractAmountsInYuan(text: string): number[] {
  // 只清理数字内部空格, 避免破坏普通文本的分词边界.
  const cleaned = text.replace(/(\d)[ \t]+(?=[\d.,])/g, "$1");
  const amountPattern = /(\d[\d,]*(?:\.\d+)?)\s*(万亿|亿|万)?/g;

  return [...cleaned.matchAll(amountPattern)].flatMap((match) => {
    const value = Number.parseFloat(match[1].replace(/,/g, ""));
    if (Number.isNaN(value)) return [];

    // 所有候选值统一到元, 后续比较不再依赖答案使用的展示单位.
    const multiplier = match[2] ? UNIT_TO_YUAN[match[2]] : 1;
    return [value * multiplier];
  });
}

export function matchesExpectedAmount(
  answer: string,
  expectedYuan: number,
  relativeTolerance = 0.005,
): boolean {
  return extractAmountsInYuan(answer).some((candidate) => {
    const relativeError =
      Math.abs(candidate - expectedYuan) / expectedYuan;
    // 0.5% 容差只处理显示精度, 不能掩盖数量级错误.
    return relativeError <= relativeTolerance;
  });
}
```

数值准确性解决的是金额和单位问题. 它不能识别业务口径. 公司总营收, 产品营收和分部营收都可能是正确数字, 但不能互相替代. 对比类问题必须额外检查两个数字是否来自相同会计口径.

![单位漂移与口径错配](./assets/04_单位漂移与口径错配分析.png)

## 用同一套题对比检索策略

评估时只改变检索器, 生成模型, prompt, top-k 和 judge 配置保持一致. 否则指标变化无法归因.

```ts
export type Mode = "dense" | "hybrid" | "rerank";

const RETRIEVERS: Record<Mode, RetrieveFn> = {
  // 3 个入口共享后续生成和评分链路, 保证实验只改变检索策略.
  dense: (query, options) => retrieve(query, options),
  hybrid: (query, options) => hybridRetrieve(query, options),
  rerank: (query, options) => rerankRetrieve(query, options),
};
```

本地 6 题小样本的聚合结果如下:

| 指标 | dense | hybrid | rerank |
| --- | ---: | ---: | ---: |
| Faithfulness | 0.542 | 0.320 | 0.612 |
| Answer relevancy | 0.595 | 0.425 | 0.622 |
| Context relevance | 0.345 | 0.312 | 0.520 |
| Numeric accuracy | 0.833 | 0.833 | 1.000 |

![三种检索策略的评估结果](./assets/02_方法比较数据仪表盘.png)

这个结果不能推出 `rerank 永远优于 hybrid`. 它只能说明, 在当前语料, 问题集和参数下, hybrid 没有解决主要瓶颈, rerank 修复了一道关键题的排序问题.

## 为什么 hybrid 可能退化

Hybrid search 通常并行执行关键词检索和向量检索, 再用 RRF 合并排名. [Azure AI Search 的 RRF 文档](https://learn.microsoft.com/en-us/azure/search/hybrid-search-ranking)说明, 文档在多个结果列表中的排名会共同贡献最终分数.

这套机制有明确价值, 但不保证每个数据集都提升. 在本次样本中, 某些关键表格块只被一路检索命中. 一些语义宽泛的说明段落却同时出现在多路结果中. RRF 奖励多路排名共识后, 后者可能把真正包含答案的表格块挤出 top-k.

结论不是放弃 hybrid, 而是先做消融实验:

1. 保存每一路检索的原始排名.
2. 记录 RRF 合并后的排名变化.
3. 对失败题检查答案块在哪一步掉出 top-k.
4. 调整各路候选数, 权重或融合参数后重跑同一评估集.

## rerank 修复的是排序, 不是召回

本次 Numeric accuracy 从 0.833 提升到 1.000, 收益集中在一道净利润题. 答案块已经存在于候选集, 但 dense 和 hybrid 没有把它排进生成模型可见的前几名. 这属于排序问题.

Retrieve and rerank 的典型做法是先用高效检索器召回候选, 再用 CrossEncoder 对 query 和候选文档成对打分. [Sentence Transformers 官方文档](https://www.sbert.net/examples/sentence_transformer/applications/retrieve_rerank/README.html)也采用两阶段结构. CrossEncoder 通常能提供更细的相关性判断, 代价是推理成本更高, 因此只适合处理第一阶段的小候选集.

诊断顺序很重要:

1. 答案块不在索引中: 修解析或入库.
2. 答案块在索引中, 但未进入候选集: 修召回.
3. 答案块进入候选集, 但未进入 top-k: 修排序或加 rerank.
4. 证据正确, 答案仍错误: 修 prompt, 生成模型或后处理.

如果问题在第 3 层, 继续增加数据或反复改 prompt 不会直接解决根因.

## LLM-as-Judge 只能做方向性信号

LLM judge 不是确定函数. 同一道题在不同运行中可能得到不同分数. 结构化输出能力也取决于真实模型 endpoint, 不能只看模型名或 SDK 类型声明.

上线前应做 3 项检查:

1. judge 与被测生成模型解耦.
2. 用真实 endpoint 验证结构化输出.
3. 对同一批样本重复评分, 记录均值和波动.

此外, judge 只能判断答案是否被上下文支持. 如果上下文本身把公司总营收和产品营收混在一起, 高 Faithfulness 仍可能对应错误业务口径.

因此, 财报问答的交付门禁至少应包含:

| 门禁 | 实现方式 | 是否确定 |
| --- | --- | --- |
| 语义一致性 | LLM judge | 否 |
| 数值准确性 | 纯代码 | 是 |
| 拒答成功率 | 固定话术或结构化状态 | 是 |
| 引用格式合规率 | Schema 或正则 | 是 |
| 业务口径正确性 | 规则加人工抽检 | 部分确定 |

![语义指标与数值门禁](./assets/05_语义三角与数值准确性分析.png)

## 可复跑的最小流程

评估脚本应把问题集, 标准答案, 期望出处和题型作为版本化数据保存. 每次修改检索, prompt 或模型后, 使用同一命令重跑:

```bash
# mode 只改变检索器, 其余评估配置保持不变.
npx tsx src/eval/run.ts dense
npx tsx src/eval/run.ts hybrid
npx tsx src/eval/run.ts rerank
```

输出中不要只保留聚合均分. 至少保存每道题的 query, retrieved chunks, answer, citation, numeric match 和 judge 原始结果. 聚合分告诉你系统是否变化, 单题证据才能解释为什么变化.

## 延伸阅读

- [Ragas: Available Metrics](https://docs.ragas.io/en/stable/concepts/metrics/available_metrics/)
- [Azure AI Search: Relevance scoring using RRF](https://learn.microsoft.com/en-us/azure/search/hybrid-search-ranking)
- [Sentence Transformers: Retrieve and Re-Rank](https://www.sbert.net/examples/sentence_transformer/applications/retrieve_rerank/README.html)
- [Sentence Transformers 官方 rerank Demo](https://github.com/huggingface/sentence-transformers/blob/main/examples/sentence_transformer/applications/retrieve_rerank/retrieve_rerank_simple_wikipedia.ipynb)

## 最后

RAG 评估的目标不是追求一个漂亮总分, 而是把失败定位到解析, 召回, 排序, 生成或业务校验中的具体一层.

语义指标适合发现方向, 确定性代码适合守住数字, 人工抽检适合处理业务口径. 三者缺一不可. 如果要复现实验, 可以从上面的 [Sentence Transformers 官方 Demo](https://github.com/huggingface/sentence-transformers/blob/main/examples/sentence_transformer/applications/retrieve_rerank/retrieve_rerank_simple_wikipedia.ipynb)搭建两阶段检索, 再把数值门禁和 no-answer 题型加入自己的评估集. 技术问题可以直接在对应开源仓库的 Issue 中讨论.
