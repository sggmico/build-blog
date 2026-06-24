---
title: RAG 向量库实战: 嵌入模型, 1024 维与 pgvector 索引设计
date: 2026-06-24
tags: [RAG, Embedding, pgvector, PostgreSQL, Vector Database]
description: 本文从中文财报 RAG 的真实链路出发, 解释嵌入模型如何把文本变成向量, 为什么要校准 1024 维, 413 报错应优先限制单块长度, 以及 pgvector 如何把向量检索和元数据过滤放进同一张表.
---

# RAG 向量库实战: 嵌入模型, 1024 维与 pgvector 索引设计

## 摘要

RAG 系统里, 生成模型很显眼, 向量层却更容易决定成败.文本切成块之后, 每个块都要先经过嵌入模型, 变成一组可计算的浮点数.随后, 这组数要和公司, 年份, 页码, 原文等元数据一起入库.检索时, 系统才能同时回答两个问题: 哪些文本意思最接近查询, 哪些文本属于指定公司和报告期.

本文基于一个中文财报 RAG 的工程链路, 讨论四个容易被忽略的问题: 嵌入到底在算什么, 1024 维应该由谁决定, 413 报错真正卡在单条文本还是批量条数, 以及为什么在中小规模场景里用 Postgres + pgvector 比直接上专用向量库更稳.

## 嵌入不是关键词匹配, 而是语义坐标

用户问 "茅台去年挣了多少钱", 年报里可能写的是 "营业总收入" 或 "归属于母公司股东的净利润".关键词匹配会在这里犯难, 因为字面并不完全相同.

嵌入模型解决的是另一个问题: 把文本映射成向量, 让语义相近的文本在向量空间里距离更近.查询时, 用户问题也会被映射成向量, 再和库里的文档块计算距离.这个过程不要求词面一致, 它要求语义距离足够可靠.

在财报场景里, 这件事尤其要紧.财报术语密集, "营业收入", "营业总收入", "归母净利润", "净利润" 之间差一点就差很多.如果嵌入模型对中文和金融表达不敏感, 后面的向量库再快, 也只是在快速返回错误内容.

## 中文财报为什么选 BGE-large-zh-v1.5

通用教程常用 `text-embedding` 类模型做演示, 但生产系统不能只看教程默认值.这里选择 `BGE-large-zh-v1.5`, 核心原因是中文财报的语义分辨率要求更高.

[BAAI/bge-large-zh-v1.5](https://huggingface.co/BAAI/bge-large-zh-v1.5) 的模型卡将它标记为中文嵌入模型, 并在 C-MTEB 表格中给出 1024 维的 embedding dimension.这个信息很关键: 维度不是业务代码拍脑袋定的, 而是模型输出决定的.

对中文财报 RAG 来说, 模型选择至少要看三件事:

1. 中文语义对齐: 模型能否稳定区分中文财务术语.
2. 成本与部署: 批量建库会调用大量嵌入请求, 延迟, 单价和内网部署能力都会进入账本.
3. 领域可扩展性: 通用中文模型只是底座, 后续如果要提升金融口径识别, 还需要领域数据微调或重排器.

这里没有神秘技巧.模型越贴近数据语言, 向量距离越不容易跑偏.

## 1024 维必须三处一致

`1024` 这个数字很容易被当成普通配置.它其实是一个三方契约:

1. 嵌入模型真实输出 1024 维.
2. 应用配置 `EMBED_DIM` 填 1024.
3. pgvector 表或索引按 1024 维创建.

任何一处不一致, 都不会温柔提醒你.常见情况是启动时没事, 写库或查询时才炸.

```ts
// 设计意图: 把维度交给环境配置, 便于换模型时同步调整.
EMBED_DIM: z.coerce.number().int().positive(),
```

这段配置只能保证 `EMBED_DIM` 是正整数, 不能证明它和模型输出一致.所以换嵌入模型时, 不应该只改模型名.正确动作是同时确认模型卡, 应用配置和数据库索引.

维度高低也有成本.高维向量通常能表达更多语义细节, 但存储更大, 相似度计算更重, 索引也更占资源.低维向量更省, 但复杂术语容易挤在一起.中文财报选择 1024 维, 不是因为数字漂亮, 而是模型和场景共同把它推到了这里.

## 413 报错: 先限制单块长度, 再谈批量

嵌入接口返回 413, 本质是请求体超过服务端可接受范围.工程上最危险的误判, 是把它简单理解成 "批量太大".

诊断脚本测出的现象是: 单条短文本可以通过, 10 条和 32 条短文本也能通过; 单条 2000 字和 8000 字会失败.也就是说, 这条链路里最先要管住的是单块长度, 不是批量条数.

```ts
// 设计意图: 用较小批量降低单次失败成本, 但它不是防 413 的主红线.
const BATCH = 10;

export async function embedTexts(texts: string[]): Promise<number[][]> {
  const out: number[][] = [];

  for (let i = 0; i < texts.length; i += BATCH) {
    // 设计意图: 每批独立请求, 失败时更容易重试和定位异常样本.
    const { embeddings } = await embedMany({
      model,
      values: texts.slice(i, i + BATCH),
    });

    out.push(...embeddings);
  }

  return out;
}
```

真正防 413 的红线应该放在切块阶段, 例如 `MAX_CHARS = 400`.`BATCH = 10` 是请求层的保守裕量, 负责降低失败面和内存压力.把这两个常量讲反, 系统会在错误的位置加固, 看似严谨, 实际漏风.

## 为什么用 pgvector, 而不是先上专用向量库

向量库选型不能脱离查询形态.财报问答里, 用户问题通常有两部分:

1. 语义条件: "营收多少", "现金流怎么样", "净利润增长了吗".
2. 结构条件: 公司, 年份, 报告期, 页码范围.

如果向量在一套系统, 元数据在另一套系统, 查询就会变成跨库协作.先向量召回再回关系库过滤, 或先过滤再去向量库找相似文本, 都会增加工程复杂度和延迟.

pgvector 的价值在这里很直接: 它让向量列和元数据列留在同一张 Postgres 表里.[pgvector 官方文档](https://github.com/pgvector/pgvector) 支持在 Postgres 中创建 vector 类型, 并提供 L2, inner product, cosine 等距离运算符.[Supabase 文档](https://supabase.com/docs/guides/database/extensions/pgvector) 也明确把 pgvector 定位为用于 embeddings 和 vector similarity 的 Postgres 扩展.

```ts
// 设计意图: 向量和元数据同批写入, 避免检索阶段再跨库拼接上下文.
const vectors = await embedTexts(texts);
const ids = metas.map((_, i) => `${company}_${year}_${i}`);

await store.upsert({
  indexName: INDEX_NAME,
  vectors,
  metadata: metas,
  ids,
});
```

`metadata` 里至少应该包含 `text`, `company`, `period`, `sourcePage`, `isTable`.这样查询时可以先把公司或报告期卡住, 再按向量距离排序.语义检索和结构过滤不是两条链, 它们从入库那一刻就应该是一行数据.

## 索引默认值不要猜, 要从活库读

很多框架会替你创建向量表和索引.方便是真的方便, 但默认值也会藏起来.

示例链路里, 初始化 SQL 只显式开启 pgvector:

```sql
/* 设计意图: 只启用 vector 扩展, 表结构和索引交给运行时创建. */
CREATE EXTENSION IF NOT EXISTS vector;
```

业务代码创建索引时只传了维度:

```ts
// 设计意图: 框架负责表和索引细节, 应用只声明索引名和向量维度.
await store.createIndex({
  indexName: INDEX_NAME,
  dimension: config.embed.dim,
});
```

这时光读代码, 只能知道系统创建了一个向量索引.它到底用 cosine, L2 还是 inner product, 用 ivfflat 还是 HNSW, 参数是多少, 都不能靠猜.

活库诊断读出的结果是:

```text
rows: 25159
dimension: 1024
vector type: vector
metric: cosine
index type: ivfflat
config: {"lists":100}
```

这组信息改变了优化方向.pgvector 文档说明, IVFFlat 会把向量划分到多个 lists, 查询时只扫描一部分最接近的 lists.它构建更快, 内存占用更低, 但在速度和召回的权衡上通常不如 HNSW.HNSW 查询性能更好, 但构建更慢, 内存需求也更高.

所以结论不是 "ivfflat 一定差" 或 "HNSW 一定好".结论是: 先确认真实索引类型, 再讨论优化.否则你以为自己在调 HNSW, 实际库里跑的是 ivfflat.

## 查询过滤的边界

pgvector 同表过滤很方便, 但不是没有边界.Supabase 文档提醒过一个重要问题: 当向量索引和 `WHERE` 条件一起使用时, 过滤可能发生在索引扫描之后, 结果数可能少于预期.

这会影响财报 RAG 的召回策略.比如你希望返回 5 个 "maotai + 2023" 的结果, 近似索引先扫到的候选里未必正好有足够多满足条件的行.数据量小的时候影响不明显, 数据量上来以后, 就要考虑以下策略:

1. 给常用过滤列建 B-tree 或组合索引.
2. 对高频过滤条件使用 partial index 或分区.
3. 调整 ivfflat lists 和 probes, 或评估 HNSW.
4. 在召回后增加重排, 避免单次近似检索决定最终答案.

这也是为什么 RAG 不能只看 "能不能返回答案".真正要看的是: 召回候选是否稳定, 过滤条件是否可靠, 索引默认值是否可解释.

## 最小验证清单

如果你也在做文档 RAG, 可以按这个顺序查向量层:

1. 模型卡: 确认嵌入维度, 语言能力和输入长度.
2. 配置: 确认 `EMBED_MODEL` 和 `EMBED_DIM` 同步变更.
3. 切块: 用脚本验证单块长度上限, 不要只调 batch.
4. 入库: 确认向量, 原文和元数据同批写入.
5. 活库: 读取真实行数, 维度, 距离度量, 索引类型和索引参数.
6. 查询: 用随机向量或小样本验证过滤条件是否生效.

```text
unfiltered hits: 3 (expect 3)
filtered company=maotai: 2 (expect 2)
SMOKE OK
```

这类冒烟测试不需要调用嵌入模型, 也不依赖外部 API key.它只验证一件事: 数据库层的 "建索引, 写入, 过滤检索" 是否真的通了.

## 延伸阅读

- [BAAI/bge-large-zh-v1.5 模型卡](https://huggingface.co/BAAI/bge-large-zh-v1.5)
- [pgvector 官方文档](https://github.com/pgvector/pgvector)
- [Supabase pgvector 文档](https://supabase.com/docs/guides/database/extensions/pgvector)

