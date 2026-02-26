# META — Contextual Retrieval

| 字段 | 内容 |
|------|------|
| **名称** | Contextual Retrieval |
| **中文名** | 上下文检索 |
| **链接** | https://www.anthropic.com/engineering/contextual-retrieval |
| **来源** | Anthropic Engineering Blog |
| **发布时间** | 2024-09-19 |
| **分类** | RAG, Retrieval, Embeddings, BM25 |
| **作者** | Anthropic Engineering Team |
| **阅读时长** | ~12 分钟 |
| **难度** | Intermediate — 适合构建 RAG 系统的工程师 |

## 摘要

本文介绍了"上下文检索"方法，通过在嵌入前为每个文本块添加上下文描述，显著提升 RAG 系统的检索精度。结合上下文嵌入、上下文 BM25 和重排序，检索失败率可降低 67%（从 5.7% 降至 1.9%）。核心思想是利用 Claude 3 Haiku 为每个文本块自动生成 50-100 token 的背景说明。

## 相关文件

- [README_EN.md](./README_EN.md) — 英文详细整理
- [README_CN.md](./README_CN.md) — 中文详细整理
