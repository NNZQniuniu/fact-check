---
name: fact-check-by-docs
description: 读一份已经产出的核查文档,接着往下核查:推进没打勾的、处理没处理的 diff。
disable-model-invocation: true
argument-hint: "核查文档的路径,或描述是哪一份"
---

找那份核查文档(用户给的路径 / 工作目录下的 `核查-*.md`),读通,按 `fact-check` 的纪律**只推进没做完的部分**:

- 自己看过、认的 → 打勾(署你的名 + 日期)。
- 不认的 → 写 diff(改前 / 改后)+ 一句为什么。
- **已有的勾不许改**;要推翻就写 diff 说理由,让下一个看的人判。
- **找不到那份文档就直说**,让用户先用 `fact-check-with-docs` 建一份;不许自己另建一份。
