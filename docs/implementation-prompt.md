# PM Server 実装プロンプト v2

## 事前確認

- Python 3.11+ が利用可能か確認（`python3 --version`）
- `pip install fastmcp pydantic pyyaml click jinja2` が実行可能か確認
- `docs/design.md` を読み、設計全体を理解すること
- 設計書に記載されたすべてのモジュールを実装すること

## 概要

Claude Code 用のプロジェクト管理 MCP Server を構築する。
**ワンコマンドインストール、ゼロ設定**で動作し、PyPI 公開を前提とする。

## タスク一覧（実装順）

### Task 1: プロジェクト構造 + pyproject.toml

設計書 §7 の構成に従い、全ディレクトリ・ファイルを作成。
pyproject.toml は設計書 §7.1 をベースに、正確な依存関係を記述。

### Task 2: models.py

設計書 §3.3 のデータモデルをすべて Pydantic v2 で実装。
- ProjectStatus, TaskStatus, Priority (Enum)
- Task, Phase, Decision, Milestone, Risk, DailyLogEntry
- Project（project.yaml のルートモデル）
- Registry（registry.yaml のルートモデル）
- バリデーション: ID形式、日付範囲、ステータス遷移

### Task 3: storage.py

YAML ファイルの CRUD。
- load_project / save_project
- load_tasks / save_tasks / add_task / update_task
- load_decisions / save_decisions / add_decision
- add_daily_log
- load_registry / save_registry / register_project / unregister_project
- ファイル未存在時はデフォルト値で自動生成
- YAML出力は human-readable（コメントヘッダー、整形済み）

### Task 4: discovery.py

設計書 §9 の全関数を実装。
- detect_project_info: Cargo.toml / package.json / pyproject.toml / git / README から推定
- discover_projects: 再帰スキャンで .pm/ 持ちプロジェクトを検出

### Task 5: installer.py

設計書 §8 の全関数を実装。
- find_claude_settings: 複数のパスから設定ファイルを検出
- install_mcp: MCP サーバー登録を注入（既存設定を壊さない）
- uninstall_mcp: 登録解除

重要: 既存の settings.json の他の設定を絶対に壊さないこと。

### Task 6: server.py

設計書 §4.1 の全17ツールを FastMCP で実装。
- 全ツールで project_path はオプション（設計書 §3.2 の自動検出ロジック）
- pm_init: .pm/ 作成 + discovery.detect_project_info + registry 自動登録
- pm_dashboard: HTML 生成（dashboard.py に委譲）
- pm_next: 優先度 × 依存関係 × フェーズ順で推薦
- pm_velocity: 日次ログの done タスク数から週次集計

### Task 7: dashboard.py + templates/

設計書 §6 のダッシュボード仕様に従い実装。
- Jinja2 テンプレート（HTML）
- Chart.js CDN（ベロシティチャート）
- ダークテーマ
- 単体ビュー + 横断ビューの2テンプレート
- テキストフォールバック（format="text" 時）

### Task 8: __main__.py (CLI)

設計書 §4.2 の CLI を click で実装。
- `pm-server install` / `uninstall`
- `pm-server serve` (MCP Server 起動)
- `pm-server discover [path]`
- `pm-server status`
- `pm-server version`

### Task 9: テスト

pytest で全モジュールをテスト。
- test_models.py: バリデーション、Enum、シリアライゼーション
- test_storage.py: YAML 往復テスト、デフォルト生成、更新
- test_server.py: 各ツールの正常系・異常系
- test_installer.py: 設定ファイル注入・解除（mock使用）
- test_discovery.py: 各種プロジェクトタイプの情報推定
- test_dashboard.py: HTML 生成の妥当性

## ガード & 制約

- 既存の Claude Code 設定（settings.json）を壊さない。キーの追加のみ。
- YAML は safe_load / safe_dump のみ使用（セキュリティ）
- タスクIDの形式: `{PROJECT_PREFIX}-{3桁連番}` (例: RVIM-001)
- 日付は ISO 8601
- パスは resolve() で絶対パスに正規化して保存
- `.pm/` は .gitignore に追加しない（git 追跡対象）
- エラーメッセージは英語（ユーザーは Claude が翻訳）

## テスト

1. `pytest tests/ -v` で全テストパス
2. `pm-server install` → settings.json に pm-server が追加される
3. `pm-server serve` → MCP Server が起動する（stdio）
4. pm_init → .pm/ 作成 + registry 自動登録
5. pm_add_task → pm_tasks で取得可能
6. pm_update_task → ステータス変更が反映
7. pm_dashboard → 有効な HTML が生成される
8. pm_discover → .pm/ 持ちプロジェクトが検出される
