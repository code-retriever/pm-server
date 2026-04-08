# PM Server 引き継ぎ資料

**Date**: 2026-04-03
**Status**: v0.1.0 インストール済み・実運用開始可能

---

## 1. 現在の状態

### インストール済み
- `pip install -e .` でeditable install完了
- `pm-server install` で Claude Code MCP設定に登録済み（`~/.claude/settings.json`）
- Claude Code 再起動で即利用可能

### コード品質
- 115テスト全パス（0.99s）
- ruff lint / format クリーン
- コードレビュー済み（Minor fixes → 全修正完了）
- 最終レビュー済み（ブロッカー3件 → 全修正完了）
- セキュリティ全項目PASS

### 構成
```
/Users/flc001/Desktop/work/Develop/00_project/01_flc-app/pm-server/
├── CLAUDE.md                    # Claude Code プロジェクト知識
├── LICENSE                      # MIT (Shinichi Nakazato / FLC design co., ltd.)
├── README.md                    # PyPI 公開用 README
├── pyproject.toml               # パッケージ定義
├── .claude/
│   ├── settings.json            # 権限設定
│   └── commands/
│       ├── implement.md         # /implement
│       ├── test.md              # /test
│       ├── lint.md              # /lint
│       ├── review.md            # /review (コードレビュー)
│       ├── final-review.md      # /final-review (リリース前レビュー)
│       └── fix-review.md        # /fix-review (レビュー修正)
├── src/pm_server/
│   ├── __init__.py
│   ├── __main__.py              # CLI (click)
│   ├── server.py                # FastMCP 15ツール
│   ├── models.py                # Pydantic v2 (12モデル, 9 Enum)
│   ├── storage.py               # YAML CRUD
│   ├── installer.py             # Claude Code MCP 自動登録
│   ├── discovery.py             # プロジェクト情報自動推定
│   ├── dashboard.py             # HTML/テキスト ダッシュボード
│   ├── velocity.py              # ベロシティ・リスク検知
│   ├── utils.py                 # パス解決・ID生成・集計
│   └── templates/
│       ├── dashboard_single.html
│       └── dashboard_portfolio.html
├── skill/SKILL.md               # PM スキル定義
├── tests/ (115テスト)
└── docs/
    ├── design.md                # 設計書 v0.2.0
    └── implementation-prompt.md
```

---

## 2. 使い方

### 初回（プロジェクトで PM を始める）
```
> PM初期化して
# または
> pm_init を実行して
```
→ `.pm/` 作成、レジストリ自動登録、プロジェクト情報推定

### 日常
```
> 進捗は？                    → pm_status
> 次にやること                 → pm_next
> タスク追加：○○を実装        → pm_add_task
> RVIM-003 完了               → pm_update_task
> ダッシュボード見せて         → pm_dashboard (HTML)
> 全プロジェクトの状態         → pm_dashboard() (横断ビュー)
> この設計にした理由を記録     → pm_add_decision
> ブロッカーある？             → pm_blockers
```

### CLI
```bash
pm-server install      # MCP登録（済み）
pm-server uninstall    # MCP解除
pm-server serve        # MCP Server起動（Claude Code が自動実行）
pm-server discover .   # プロジェクト検出
pm-server status       # ステータス表示
```

---

## 3. 更新方法

### コード修正後
```bash
# editable install なので、コード修正後は Claude Code 再起動だけで反映
# pip install し直す必要なし
```

### pyproject.toml の依存関係変更後
```bash
cd /Users/flc001/Desktop/work/Develop/00_project/01_flc-app/pm-server
pip install -e .
# Claude Code 再起動
```

---

## 4. 残タスク（GitHub公開前）

### 必須
- [ ] docs/design.md の Author を "Shinichi Nakazato / FLC design co., ltd." に修正（5行目）
- [ ] 実運用テスト（Rvim + 既存プロジェクトで2週間使う）
- [ ] 実運用フィードバックを反映

### 推奨
- [ ] CHANGELOG.md 作成
- [ ] .github/workflows/test.yml (pytest CI)
- [ ] .github/workflows/publish.yml (PyPI 自動公開)
- [ ] README.md にスクリーンショット追加
- [ ] GitHub リポジトリ作成・初回push

### 将来
- [ ] git hook 連携（コミット時にタスク自動更新）
- [ ] サブエージェント pm-check（セッション開始/終了フック）
- [ ] Skill の Claude Code 正式スキルとしてのインストール

---

## 5. アーキテクチャ要点

### データフロー
```
Claude Code → MCP (stdio) → pm-server serve → storage.py → .pm/*.yaml
                                            → dashboard.py → HTML
                                            → velocity.py → 分析
```

### グローバルレジストリ
```
~/.pm/registry.yaml  ← 全プロジェクトの登録簿
  ↓ 参照
project-A/.pm/       ← 各プロジェクト固有データ
project-B/.pm/
```

### 設計原則
1. Zero Configuration — pip install + pm-server install で完了
2. Auto-everything — 登録・検出・推定は全自動
3. Git-friendly — plain text YAML
4. Plugin-first ではなく MCP-first — Claude Code のツールとして動作

---

## 6. トラブルシューティング

### pm-server コマンドが見つからない
```bash
# pyenv の Python パスを確認
pyenv which pip
pip install -e .
```

### MCP Server が応答しない
```bash
# 直接起動してテスト
pm-server serve
# Ctrl+C で終了

# Claude Code の設定を確認
cat ~/.claude/settings.json | python3 -m json.tool
```

### .pm/ が壊れた
```bash
# 該当ファイルだけ削除すれば自動再生成
rm project-path/.pm/tasks.yaml  # 次回 pm_tasks 時に空で再生成
```

---

## 7. 関連ファイル

| ファイル | 場所 |
|---|---|
| MCP設定 | `~/.claude/settings.json` |
| グローバルレジストリ | `~/.pm/registry.yaml` |
| Python環境 | pyenv 3.13.2 (`~/.pyenv/versions/3.13.2/`) |
| editable install | `pm-server/src/pm_server/` を直接参照 |
