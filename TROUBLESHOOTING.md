# Troubleshooting

GitHub Agentic Workflows（Public Preview）のセットアップ・動作確認で遭遇した問題と対処法。

## セットアップ

### `gh aw` が「not authenticated」になる

**原因**: GitHub CLI の認証に `workflow` スコープが含まれていない。

```bash
# スコープを追加して再認証
gh auth login --scopes repo,workflow
```

### `gh aw add-wizard` が動かない（Windows / Git Bash）

**原因**: ウィザードは Linux / macOS / WSL のみ対応。Windows の Git Bash では動作しない。

**対処**: `.github/workflows/` に Markdown ファイルを手動で作成する。フォーマットは `issue-triage.md` / `ci-doctor.md` を参照。

### `gh aw compile` で `issues: write` がエラーになる

```
error: strict mode: write permission 'issues: write' is not allowed for security reasons.
Use 'safe-outputs.create-issue', 'safe-outputs.add-comment', or 'safe-outputs.update-issue'
to perform write operations safely.
```

**原因**: Agentic Workflows は直接の write 権限を許可しない。代わりに `safe-outputs` を使う。

**対処**: permissions は `read` のまま、frontmatter に `safe-outputs` を追加する。

```yaml
permissions:
  issues: read          # write ではなく read

safe-outputs:           # ここで書き込み操作を許可
  add-labels:
    allowed: [bug, feature]
    max: 3
```

使える safe-outputs: `create-issue`, `create-pull-request`, `add-comment`, `update-issue`, `add-labels`, `remove-labels`

注意: `update-issue` はタイトル・本文・ステータスの更新のみ。ラベル操作には `add-labels` / `remove-labels` を使う。

## 実行時

### activation ステップの「Upload activation artifact」で失敗する

```
Error: Create Artifact Container failed: The artifact name activation is not valid.
```

**原因**: `actions/upload-artifact@v4` の既知バグ（[Issue #490](https://github.com/actions/upload-artifact/issues/490)、オープン・未修正）。クライアント側で「Artifact name is valid!」と表示された直後に API が 400 を返す。GitHub サーバー側の断続的な問題で、ステータスページにインシデントとして報告されないことが多い。

Re-run 時は同一 run_id 内でアーティファクト名 `activation` が衝突し、確実に失敗する。

gh-aw の audit によると成功率は約 89%（10回に1回は失敗する）。Public Preview の現時点では不安定さが残る。

**対処**:
- **Re-run は使わない**。新しい issue / push で新規 run をトリガーする
- 新規 run でも失敗する場合は GitHub 側の一時的なインフラ問題。数時間おいて再試行
- `gh aw audit <run-id>` でデバッグ情報を確認できる
- 解消しない場合は [github/gh-aw](https://github.com/github/gh-aw/issues) に Issue 報告

**関連 Issue**:
- [actions/upload-artifact #490](https://github.com/actions/upload-artifact/issues/490) — 断続的な「artifact name is not valid」400 エラー（オープン）
- [actions/upload-artifact #410](https://github.com/actions/upload-artifact/issues/410) — intermittent 400 エラー
- [actions/upload-artifact #478](https://github.com/actions/upload-artifact/issues/478) — 同名アーティファクトの衝突
- [github/gh-aw #29499](https://github.com/github/gh-aw/issues/29499) — `include-hidden-files` 不足（修正済み）

### Ci Doctor が CI 成功時にも起動して失敗する

**原因**: `workflow_run` トリガーは成功・失敗どちらでも発火する。AI が「何もしない」と判断するが、safe-output のツール呼び出し要件を満たせず失敗することがある。

**対処**: 想定内の挙動。CI が失敗したときだけ意味のあるコメントがつく。

## 現在の状況（2026-06-27）

- **Issue Triage**: activation の artifact upload エラーで動作未確認
- **Ci Doctor**: 同上
- **次のアクション**: 時間をおいて新しい issue で再テスト
