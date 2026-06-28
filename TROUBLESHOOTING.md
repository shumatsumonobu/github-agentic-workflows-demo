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

**原因**: `gh aw compile` が生成する `.lock.yml` が `actions/upload-artifact@v3.2.2` を使っている。v3 の artifact API は一部のリポで 400 エラーを返す。

**対処**: `.lock.yml` 内の `upload-artifact` と `download-artifact` を v4 に書き換える。

```bash
# issue-triage.lock.yml と ci-doctor.lock.yml の両方で実行
# upload-artifact を v4 に
sed -i 's|actions/upload-artifact@c6a366c94c3e0affe28c06c8df20a878f24da3cf # v3.2.2|actions/upload-artifact@v4|g' .github/workflows/*.lock.yml

# download-artifact も v4 に（v3 と v4 の artifact API に互換性がないため）
sed -i 's|actions/download-artifact@a9bc5e6ef2cb54c177f32aa5726adaa15e7e2d59 # v3.1.0|actions/download-artifact@v4|g' .github/workflows/*.lock.yml
```

注意:
- `.lock.yml` は「DO NOT EDIT」と書かれているが、v3 → v4 の変更は動作に問題なし（検証済み）
- `gh aw compile` を再実行すると v3 に戻るので、compile 後に毎回この置換が必要
- gh-aw 側のバグ。将来のバージョンで修正される可能性あり

**関連 Issue**:
- [actions/upload-artifact #490](https://github.com/actions/upload-artifact/issues/490) — 断続的な「artifact name is not valid」400 エラー（オープン）
- [actions/upload-artifact #410](https://github.com/actions/upload-artifact/issues/410) — intermittent 400 エラー
- [github/gh-aw #29499](https://github.com/github/gh-aw/issues/29499) — `include-hidden-files` 不足（修正済み）

### Gemini 無料枠のレートリミット

```
Quota exceeded for metric: generate_content_free_tier_requests
limit: 20, model: gemini-3-flash
```

**原因**: Gemini 無料枠は 1 日あたりのリクエスト数に制限がある（gemini-3-flash で 20 回/日）。テストを繰り返すと使い切る。

**対処**: 翌日にリセットされるので待つ。有料プランにアップグレードすれば制限が緩和される。

### Ci Doctor が CI 成功時にも起動して失敗する

**原因**: `workflow_run` トリガーは成功・失敗どちらでも発火する。AI が「何もしない」と判断するが、safe-output のツール呼び出し要件を満たせず失敗することがある。

**対処**: 想定内の挙動。CI が失敗したときだけ意味のあるコメントがつく。

## 現在の状況（2026-06-28）

- **upload-artifact v3→v4 修正**: lock ファイルを手動修正済み。activation ステップは突破確認
- **Issue Triage**: activation 通過 → agent 起動 → Gemini の無料枠レートリミットで失敗。翌日リセット後に再テスト
- **Ci Doctor**: 同上（activation は同じ修正で通過するはず）
- **次のアクション**: Gemini のクォータリセット後に新しい issue で再テスト
