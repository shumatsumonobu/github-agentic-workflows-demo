# GitHub Agentic Workflows Demo

[GitHub Agentic Workflows](https://github.blog/changelog/2026-06-11-github-agentic-workflows-is-now-in-public-preview/) を試すためのデモリポジトリ

- **GitHub Actions** — リポジトリの自動化を YAML で定義する仕組み
  1. コードがプッシュされたらテストを走らせる
  2. PR が来たらビルドする
  3. issue が立ったら担当者をアサインする
- **課題** — YAML の記法を覚える必要があり、ロジックも自分で書く
- **Agentic Workflows** — やりたいことを日本語で書くだけで AI が実行してくれる
  - YAML もロジックも書かなくていい
  - 例: 「issue が来たらラベルをつけて」と書くだけ

## このデモで試せること

`.github/workflows/` に以下の2つの Agentic Workflow（Markdown ファイル）を追加して試す

| ファイル | 何が起きるか |
|---------|------------|
| **issue-triage.md** | issue が作られたらラベルを自動付与（bug / feature / question / documentation） |
| **ci-doctor.md** | CI が失敗したらログを読んで原因をコメント |

## コスト

- **GitHub Actions**: パブリックリポなので無料
- **AI 推論（Gemini）**: Google AI Studio の無料枠で動く

## セットアップ

### 前提

- [GitHub CLI](https://cli.github.com/) がインストール済み
- [Google AI Studio](https://aistudio.google.com/apikey) で Gemini API キーを発行済み

### 手順

```bash
# 1. gh-aw 拡張をインストール
gh extension install github/gh-aw

# 2. Agentic Workflows の設定ファイルを追加（git init とは別物）
gh aw init

# 3. Gemini API キーをリポのシークレットに登録
gh secret set GEMINI_API_KEY

# 4. ワークフローを追加（ウィザードで）
gh aw add-wizard githubnext/agentics/issue-triage
gh aw add-wizard githubnext/agentics/ci-doctor

# 5. .github/workflows/*.md を読んで .lock.yml（YAML）を自動生成
gh aw compile

# 6. コミット & プッシュ
git add .github/workflows/
git commit -m "Add agentic workflows"
git push
```

### ワークフローの追加方法

手順 #4 はウィザードを使ったが、手動で書くこともできる

| 方法 | やり方 | 向いてる人 |
|------|--------|-----------|
| **ウィザード** | `gh aw add-wizard githubnext/agentics/issue-triage` | 初めての人。対話形式でエンジン選択やシークレット設定まで案内してくれる |
| **手動** | `.github/workflows/xxx.md` を自分で書く | 自由にカスタマイズしたい人 |

どちらも最終的に `.github/workflows/` に Markdown + `.lock.yml` ができる。入口が違うだけ。

API キーはリポのシークレットに保存する（手順 #3）。Markdown 側は `engine: gemini` と書くだけで、実行時に自動で繋がる。キーがコードに載ることはない。

## 仕組み

セットアップで実行したコマンドの流れはこう。

```
Markdown で定義 → gh aw compile → YAML に変換 → GitHub Actions が実行
```

### AI はどこで使われるのか

AI は「YAML を書いてくれる」わけではない。**ワークフローが動くとき、人間の代わりに判断してくれる**。

| | 従来の Actions | Agentic Workflows |
|---|---|---|
| **誰が判断するか** | ルール通りに機械的に処理 | AI が内容を読んで自分で判断 |
| **例** | 正規表現でログを grep して該当行を貼る | ログ全体を読んで原因と修正案をコメント |

### 書き方の比較

従来の YAML

```yaml
on:
  issues:
    types: [opened]
jobs:
  triage:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/github-script@v7
        with:
          script: |
            const { title, body } = context.payload.issue;
            const text = `${title} ${body}`.toLowerCase();
            const labels = [];
            if (text.includes('bug') || text.includes('error')) labels.push('bug');
            if (text.includes('feature')) labels.push('feature');
            if (!labels.length) labels.push('question');
            await github.rest.issues.addLabels({ ...context.repo, issue_number: context.issue.number, labels });
```

Agentic Workflows の Markdown

```markdown
---
on:
  issues:
    types: [opened]
engine:
  id: gemini
---

新しい issue の内容を読んで、適切なラベルをつけてください
- bug: バグ報告
- feature: 機能リクエスト
- question: 質問
- documentation: ドキュメント関連
```

## リポの中身

```
├── package.json
├── src/
│   ├── utils.js          ← add / subtract / multiply / divide の4関数
│   └── utils.test.js     ← vitest で6テスト
└── .github/workflows/
    └── ci.yml            ← push / PR 時にテスト実行
```

シンプルな Node.js プロジェクト + CI だけの最小構成

issue トリアージは issue を立てるだけで試せる。CI 失敗分析は `utils.js` を壊してプッシュするだけ。

```bash
npm install
npm test
```

## Author

**週末ものづくり部** — [@shumatsumonobu](https://x.com/shumatsumonobu)

## License

[MIT](LICENSE)
