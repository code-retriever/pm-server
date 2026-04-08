# pm-server 公開前 最終作業手順

## ステップ 1: PmAgentError → PmServerError（Claude Code で実行）

以下を Claude Code にコピペ:

````
# PmAgentError → PmServerError リネーム

## 事前確認
- `pytest` が全パスすること
- `grep -rn "PmAgentError" src/ tests/` で現在の参照箇所を確認

## タスク

### 1. models.py のクラス名変更

`src/pm_server/models.py` で:

```python
# 変更前
class PmAgentError(Exception):

# 変更後
class PmServerError(Exception):
```

`PmAgentError` を継承しているサブクラスも全て `PmServerError` に:
- `ProjectNotFoundError(PmAgentError)` → `ProjectNotFoundError(PmServerError)`
- `TaskNotFoundError(PmAgentError)` → `TaskNotFoundError(PmServerError)`
- `DecisionNotFoundError(PmAgentError)` → `DecisionNotFoundError(PmServerError)`

### 2. 全ファイルの import / 参照を一括置換

`src/pm_server/` と `tests/` 配下の全 .py ファイルで:
- `PmAgentError` → `PmServerError`

### 3. __init__.py の export 確認

`src/pm_server/__init__.py` で `PmServerError` がエクスポートされていることを確認（もし `PmAgentError` が export されていたら置換）。

## ガード
- クラスの動作・継承構造は変更しない（名前のみ変更）
- まだ外部ユーザーはいないので後方互換エイリアスは不要

## テスト
1. `ruff check src/ tests/`
2. `pytest` で全テストパス
3. `grep -rn "PmAgentError" src/ tests/` で残存がないこと
````

---

## ステップ 2: プロジェクトフォルダリネーム（ターミナルで手動実行）

ステップ 1 完了後、**ターミナル**で以下を順に実行:

```bash
# 1. 現在の editable install を解除
pip uninstall pm-server -y

# 2. フォルダリネーム
cd /Users/flc001/Desktop/work/Develop/00_project/01_flc-app/
mv pm-agent pm-server

# 3. 新しいパスで再インストール
cd pm-server
pip install -e .

# 4. 動作確認
pm-server --version
pm-server --help
```

---

## ステップ 3: 各プロジェクトで migrate 実行

```bash
# MCP 登録の切り替え
pm-server migrate
```

これで:
- 旧 `pm-agent` の MCP 登録が解除される
- 新 `pm-server` が MCP サーバーとして登録される
- 各プロジェクトの CLAUDE.md に `pm-agent` 言及があれば警告される

警告が出たプロジェクトの CLAUDE.md は手動で `pm-agent` → `pm-server` に更新:
- rvim
- ZenResona
- SynapticKnowledge
- 切替ちゃん

Claude Code を再起動して、`/mcp` で `pm-server` が表示されることを確認。

---

## ステップ 4: Git コミット整理

Claude Code で:

````
Git コミットを整理してください。
以下の方針で:

1. まず `git log --oneline` で現在のコミット履歴を確認
2. 意味のある単位でコミットをまとめる（squash）
3. コミットメッセージの方針:
   - feat: 新機能
   - fix: バグ修正
   - refactor: リネーム等
   - docs: ドキュメント
   - test: テスト
   - chore: CI/設定

理想的なコミット構成:
- feat: initial implementation of pm-server (15 MCP tools, CLI, dashboard)
- test: add comprehensive test suite (116 tests)
- fix: installer.py use claude mcp add, template path, YAML error handling
- refactor: rename package from pm-agent to pm-server
- docs: add README (EN/JA), update design doc to v0.3.0

対話的に進めて、rebase する前に計画を見せてください。
````

---

## ステップ 5: GitHub アップ前の最終チェック

Claude Code で:

````
GitHub 公開前の最終チェックを実行してください。

## チェックリスト

### コード品質
1. `ruff check src/ tests/` — エラーなし
2. `ruff format --check src/ tests/` — フォーマット済み
3. `pytest -v` — 全テストパス

### 残存チェック
4. `grep -rn "pm_agent\|pm-agent\|PmAgentError" src/ tests/ skill/ CLAUDE.md pyproject.toml` 
   — 意図的な残存（migrate 関数内、CHANGELOG 内）以外がないこと
5. `grep -rn "nakashin" .` — 旧 GitHub URL が残っていないこと

### ファイル確認
6. README.md — 英語版の内容が正しいこと
7. README.ja.md — 日本語版が存在し内容が正しいこと
8. LICENSE — MIT、著者名が正しいこと
9. CHANGELOG.md — v0.2.0 エントリが存在すること
10. pyproject.toml — name="pm-server"、URLs が code-retriever/pm-server であること

### 機密情報
11. `.gitignore` に `.venv/`, `__pycache__/`, `.pm/`, `.pytest_cache/` が含まれていること
12. `git ls-files` にハードコードされたパス（/Users/flc001/...）を含むファイルがないこと

### パッケージ動作
13. `pm-server --version` → 0.2.0
14. `pm-server --help` → install, uninstall, serve, discover, status, migrate の 6 コマンド

結果を一覧で表示してください。
````

---

## ステップ 6: GitHub Push

全チェック通過後:

```bash
cd /Users/flc001/Desktop/work/Develop/00_project/01_flc-app/pm-server
git remote add origin git@github.com:code-retriever/pm-server.git
# もしくは既に追加済みなら
git remote set-url origin git@github.com:code-retriever/pm-server.git
git branch -M main
git push -u origin main
```
