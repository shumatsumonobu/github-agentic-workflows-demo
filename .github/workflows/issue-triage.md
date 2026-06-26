---
description: "新しい issue を自動トリアージする"

on:
  issues:
    types: [opened]

permissions:
  contents: read
  issues: read
  pull-requests: read

engine:
  id: gemini

tools:
  github:
    toolsets: [default]

safe-outputs:
  add-labels:
    allowed: [bug, feature, question, documentation]
    max: 3
---

新しい issue の内容を読んで、適切なラベルをつけてください。

- bug: バグ報告
- feature: 機能リクエスト
- question: 質問
- documentation: ドキュメント関連

ラベルは1つだけ選んでください。
