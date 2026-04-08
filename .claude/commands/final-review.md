# PM Agent リリース前最終レビュー

リリース前の最終品質チェック。コードレビュー（/review）は完了済みの前提で、
パッケージング・UX・ドキュメント・セキュリティの観点でリリース可否を判定する。

---

## 1. パッケージング検証

実際に pip install して動作するかを検証する。

```bash
# クリーンな仮想環境で検証
python3 -m venv /tmp/pm-agent-test-env
source /tmp/pm-agent-test-env/bin/activate
pip install -e .
```

### チェック項目
- [ ] `pip install -e .` がエラーなく完了する
- [ ] `pm-agent --version` でバージョンが表示される
- [ ] `pm-agent install` が成功し、`~/.claude/settings.json` に `pm-agent` が追加される
- [ ] `pm-agent serve` が起動し、stdin/stdout で応答する（Ctrl+Cで終了）
- [ ] `pm-agent uninstall` が成功し、設定から `pm-agent` が除去される
- [ ] `pm-agent install` 再実行で「already registered」メッセージが出る（冪等性）
- [ ] テンプレートファイルがパッケージに含まれている:
  ```python
  python3 -c "from pm_agent.dashboard import TEMPLATES_DIR; print(TEMPLATES_DIR); assert TEMPLATES_DIR.exists()"
  ```

---

## 2. MCP Server 動作検証

FastMCP の stdio 通信が正常に動作するかを検証。

```bash
# MCP Inspector で検証（利用可能な場合）
npx @modelcontextprotocol/inspector pm-agent serve
```

または手動で:

```bash
echo '{"jsonrpc":"2.0","method":"tools/list","id":1}' | pm-agent serve
```

### チェック項目
- [ ] ツール一覧（tools/list）が全15ツールを返す
- [ ] 各ツールの description が存在し、日本語混在でも正常
- [ ] pm_init が新規ディレクトリで動作する
- [ ] pm_status が初期化済みプロジェクトで動作する

---

## 3. E2E シナリオテスト

実際の使用フローをシミュレートして検証。tmp ディレクトリで実施すること。

```bash
mkdir -p /tmp/pm-test-project
cd /tmp/pm-test-project
git init
echo '{"name":"test-project","version":"1.0.0"}' > package.json
echo "# Test Project" > README.md
```

### シナリオ A: プロジェクト初期化
1. pm_init を呼ぶ
2. .pm/ が作成される
3. project.yaml に package.json から推定した名前・バージョンが入る
4. ~/.pm/registry.yaml にパスが登録される
5. 再度 pm_init → 既存 project.yaml が上書きされない（冪等性）

### シナリオ B: タスク管理
1. pm_add_task で3件追加
2. pm_tasks でフィルタ（status=todo, priority=P0）
3. pm_update_task で1件を in_progress に変更
4. pm_update_task で1件を done に変更
5. pm_next で推薦タスクが正しい
6. pm_status で集計が正しい

### シナリオ C: ADR
1. pm_add_decision でADRを追加
2. decisions.yaml に正しく保存される
3. pm_status の出力に ADR 数が含まれる

### シナリオ D: ダッシュボード
1. pm_dashboard(format="html") で HTML が生成される
2. HTML をブラウザで開いて表示が正常（Chart.js のロード、ダークテーマ）
3. pm_dashboard(format="text") でテキスト版が生成される
4. pm_dashboard() （project_path なし）で横断ビューが生成される

### シナリオ E: ディスカバリー
1. pm_discover("/tmp") で先ほどのプロジェクトが検出される
2. pm_list で登録プロジェクトが表示される
3. テストプロジェクトを削除
4. pm_cleanup で存在しないパスが検出される

### シナリオ F: エラーケース
1. .pm/ がないディレクトリで pm_status → 明確なエラーメッセージ
2. 存在しないタスクID で pm_update_task → TaskNotFoundError
3. 壊れた YAML を .pm/tasks.yaml に書き込み → PmAgentError（クラッシュしない）

---

## 4. ドキュメント検証

### README.md
- [ ] プロジェクト概要が明確
- [ ] インストール手順が記載
- [ ] 使い方の例が記載
- [ ] スクリーンショット or ASCII デモ（なくてもOK、GitHub公開時に追加）

### pyproject.toml
- [ ] name, version, description が正確
- [ ] author 情報が正しい
- [ ] dependencies が過不足ない
- [ ] project.scripts で pm-agent が定義されている
- [ ] requires-python >= 3.11

### CLAUDE.md
- [ ] コーディング規約が記載
- [ ] ディレクトリ構成が現在の実態と一致
- [ ] 実装優先順位が現状を反映

---

## 5. セキュリティ最終チェック

- [ ] API キー・シークレットがコードに含まれていない
- [ ] .gitignore が適切（__pycache__, .venv, .env 等）
- [ ] YAML は safe_load のみ（再確認: `grep -r "yaml.load\|yaml.unsafe" src/`)
- [ ] subprocess に shell=True がない（再確認: `grep -r "shell=True" src/`)
- [ ] ファイルパスが resolve() で正規化されている

---

## 6. 出力形式

```
## リリース前最終レビュー結果

### パッケージング: PASS / FAIL
### MCP Server: PASS / FAIL
### E2E テスト: PASS / FAIL (シナリオ A-F の結果)
### ドキュメント: PASS / FAIL
### セキュリティ: PASS / FAIL

### 未解決の問題
1. ...

### リリース判定: GO / NO-GO
### 理由: ...
```
