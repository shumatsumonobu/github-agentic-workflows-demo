---
description: "CI 失敗時にログを分析して原因をコメントする"

on:
  workflow_run:
    workflows: ["CI"]
    types: [completed]
    branches: [main]

permissions:
  contents: read
  issues: read
  pull-requests: read

engine:
  id: gemini

tools:
  github:
    toolsets: [default]
---

CI が失敗したとき、ワークフローのログを読んで以下をコメントしてください。

1. 失敗の原因（どのテストが落ちたか、何のエラーか）
2. 修正案（具体的なコード変更の提案）

CI が成功した場合は何もしないでください。
