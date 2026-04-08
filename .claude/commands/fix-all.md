PM Agent の既知問題を全て修正してください。

## 修正項目（優先順）

### 1. registry 汚染の修正（最重要）

テストが `~/.pm/registry.yaml` を書き換えてしまう問題。

**修正方法**: `conftest.py` で `HOME` 環境変数と registry パスを一時ディレクトリにモック。

```python
# tests/conftest.py に追加

import os
import pytest
from pathlib import Path

@pytest.fixture(autouse=True)
def isolated_registry(tmp_path, monkeypatch):
    """全テストでグローバル registry を一時ディレクトリに隔離する。"""
    fake_home = tmp_path / "home"
    fake_home.mkdir()
    fake_pm = fake_home / ".pm"
    fake_pm.mkdir()
    
    # HOME を偽装
    monkeypatch.setenv("HOME", str(fake_home))
    
    # pm_agent 内部で Path.home() を使っている場合のパッチ
    monkeypatch.setattr(Path, "home", lambda: fake_home)
    
    # もし pm_agent が独自のグローバルパスを持っている場合
    # （storage.py や utils.py の REGISTRY_PATH 等）
    # そのモジュール変数もパッチする
    
    yield fake_home
```

**確認**: 
- テスト前後で `~/.pm/registry.yaml` が変化しないこと
- 全115テストがパスすること

### 2. 現在の registry からテスト残骸を除去

```python
# pm_cleanup を実行して、/private/var/folders/ で始まるパスを全除去
# または手動で ~/.pm/registry.yaml を編集
```

修正後に一度 `pm_cleanup` を呼び出すか、
registry.yaml を直接編集して `/private/var/folders/` のエントリを削除。

### 3. docs/design.md の Author 修正

```
旧: **Author**: Nakashin / FLC design
新: **Author**: Shinichi Nakazato / FLC design co., ltd.
```

5行目を修正するだけ。

### 4. Git コミット整理

現在のコミット状態を確認し、以下の形に整理：

```
feat: implement PM Agent MCP server with 15 tools
test: add 115 tests for all modules
fix: use claude mcp add for installer (subprocess method)
fix: correct highlight query for tree-sitter-rust 0.23
docs: add workflow guide and project status
chore: add Claude Code commands and settings
```

各コミットで `pytest` が通ること。

### 5. CHANGELOG.md 作成

```markdown
# Changelog

## [0.1.0] - 2026-04-07

### Added
- 15 MCP tools for project management
- YAML-based task, decision, and log storage
- HTML dashboard with Chart.js (single + portfolio view)
- Text dashboard fallback
- Velocity tracking and risk detection
- Project discovery and auto-registration
- CLI interface (install, uninstall, serve, discover, status)
- Claude Code integration via `claude mcp add --scope user`

### Fixed
- installer.py: use `claude mcp add` instead of writing to wrong settings file
- Template path resolution for packaged installations

### Documentation
- Development workflow guide (docs/workflow.md)
- Design document (docs/design.md)
- Project status report (docs/status.md)
```

## テスト実行

```bash
# 修正後に全テスト実行
pytest tests/ -v

# registry が汚染されていないことを確認
cat ~/.pm/registry.yaml
# → テスト用パス (/private/var/folders/...) が含まれていないこと

# ruff チェック
ruff check src/ tests/
ruff format src/ tests/ --check
```

## 完了条件

1. `pytest tests/ -v` — 全テストパス
2. `~/.pm/registry.yaml` にテスト残骸がないこと
3. `ruff check` + `ruff format --check` がクリーン
4. docs/design.md の Author が正しいこと
5. CHANGELOG.md が存在すること
6. git log が整理されていること
