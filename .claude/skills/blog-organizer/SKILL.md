---
name: blog-organizer
description: Use when given one or more blog/article URLs to fetch, analyze, and organize into structured folders with bilingual (Chinese + English) documentation, mind maps, and beginner-friendly summaries.
---

# Blog Organizer

Fetch, analyze, and organize technical blog posts into structured bilingual documentation with mind maps and beginner-friendly explanations.

## Usage

```
/blog-organizer <URL> [URL2] [URL3] ...
/blog-organizer https://example.com/article
```

Supports single or multiple URLs. Multiple URLs are processed in parallel.

## Output Structure

For each blog URL, create a folder named after the URL slug:

```
<blog-slug>/
├── META.md        # Source metadata
├── README_EN.md   # Full English notes
└── README_CN.md   # Full Chinese notes
```

**Folder naming:** Use the URL path's last segment (e.g., `building-effective-agents` from `.../engineering/building-effective-agents`).

---

## Step 1: Fetch Content

Use `WebFetch` to retrieve each URL. If multiple URLs are given, fetch them **in parallel** using simultaneous tool calls.

Prompt for WebFetch:
> "Return ALL the article text in full detail: every section, subsection, code example, table, and key point. Return as much detail as possible."

---

## Step 2: Create META.md

```markdown
# META — {Article Title}

| 字段 | 内容 |
|------|------|
| **名称** | {English title} |
| **中文名** | {Chinese title} |
| **链接** | {URL} |
| **来源** | {Source name} |
| **发布时间** | {Publish date} |
| **分类** | {Category tags} |
| **作者** | {Author(s)} |
| **阅读时长** | ~{N} 分钟 |
| **难度** | {Beginner/Intermediate/Advanced + audience note} |

## 摘要

{2-3 sentence Chinese summary of core thesis}

## 相关文件

- [README_EN.md](./README_EN.md) — 英文详细整理
- [README_CN.md](./README_CN.md) — 中文详细整理
```

---

## Step 3: Create README_EN.md

Structure every English README with these sections in order:

### 3.1 Header
```markdown
# {Article Title}

> Source: {URL}
> Published: {date} | {Source}
```

### 3.2 Overview Mind Map (ASCII)
Use ASCII tree format. Cover ALL major topics from the article:
```
Article Title
├── Section A
│   ├── Sub-topic 1
│   └── Sub-topic 2
├── Section B
│   └── ...
└── Section C
```

### 3.3 Content Sections
For each major section/concept in the article:

- **Numbered heading** matching article structure
- **What it is** — 1-2 sentence plain-language explanation
- **ASCII diagram or comparison table** for any process, flow, or distinction
- **Code blocks** — preserve original code examples verbatim
- **"Use when" / "avoid when"** lists for any pattern or tool
- **Concrete examples** drawn from the original article

### 3.4 Quick Reference (end of file)
A summary table or bullet list of the most important takeaways.

**Writing rules for English README:**
- Explain concepts as if the reader is a smart beginner
- Prefer diagrams over paragraphs for processes
- Include ALL major content — don't summarize away key details
- Preserve original code examples exactly

---

## Step 4: Create README_CN.md

Same structure as README_EN.md, but:

- All explanatory text in **Chinese**
- Section headings in **Chinese**
- Mind map labels in **Chinese**
- Code comments/variables may stay in English
- Add **"形象比喻帮助记忆"** section at the end of each major concept
- Use everyday Chinese analogies to explain technical concepts

**Writing rules for Chinese README:**
- 通俗易懂，避免术语堆砌
- 每个核心概念附上生活化类比
- ASCII 思维导图的标签用中文
- 保留代码示例不翻译
- 结尾提供"形象比喻帮助记忆"小节

---

## Diagram Conventions

**ASCII mind map (overview):**
```
Topic
├── Branch A
│   ├── Sub 1
│   └── Sub 2
└── Branch B
```

**ASCII flow diagram (processes):**
```
Input → [Step 1] → [Step 2] → Output
                      ↓
                   [Branch]
```

**Comparison table (A vs B):**
| | Option A | Option B |
|---|---|---|
| Speed | Fast | Slow |

**Box diagram (architecture):**
```
┌──────────────┐
│   Component  │
│  ┌────────┐  │
│  │ Inner  │  │
│  └────────┘  │
└──────────────┘
```

---

## Quality Checklist

Before finishing each blog folder:

- [ ] Folder name matches URL slug exactly
- [ ] META.md has all fields filled
- [ ] README_EN.md has overview mind map
- [ ] README_EN.md covers ALL major sections of the article
- [ ] README_CN.md has Chinese mind map
- [ ] README_CN.md has analogies (形象比喻) for each major concept
- [ ] Code examples preserved verbatim
- [ ] Performance numbers/benchmarks from original included
- [ ] "When to use / when to avoid" covered for any pattern

---

## Parallelism Rule

When organizing **multiple URLs**:
1. Fetch ALL URLs in parallel (single message, multiple WebFetch calls)
2. Create all folders simultaneously
3. Write all META.md files in parallel
4. Write all README_EN.md files in parallel
5. Write all README_CN.md files in parallel

Do NOT process one blog at a time sequentially — maximize parallel writes.
