---
name: fact-check-with-docs
description: 对一段话或一份材料做一次事实核查,并落一份核查文档(能交给别的 agent 接着做)。
disable-model-invocation: true
argument-hint: "要核查的内容,或材料路径"
---

跑一次 `fact-check`,走**文档模式**:

- 在工作目录下建一份 `核查-<主题>.md`(用户给了路径就用用户的)。
- 先把**声明清单**写进文档,再逐条追依据链、逐条落状态(§一、§三)。
- 每条依据写清「看了哪里」+「逐字原文」,再打勾(§七)。
- 已经有同一份文档时**不要另建** —— 接着它往下做,等同 `fact-check-by-docs`。
