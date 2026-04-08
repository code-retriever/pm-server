# pm-agent → pm-server リネーム + migrate コマンド実装プロンプト

以下を Claude Code にコピペしてください:

---

````
# pm-agent → pm-server パッケージリネーム

このプロジェクトは `pm-agent` という名前で実装済みだが、PyPI で `PMAgent` が既に存在するため `pm-server` にリネームする。
`.pm/` ディレクトリのデータ構造、MCP ツール名（`pm_init`, `pm_status` 等）は変更しない。

## 事前確認

- `src/pm_agent/` ディレクトリが存在すること
- `pyproject.toml` の `[project]` name が `"pm-agent"` であること
- `pytest` が全パスすること（現在の状態を確認）
- `.git` で未コミットの変更がないこと（あればコミットしてから開始）

## タスク一覧

### タスク 1: ディレクトリリネーム

`src/pm_agent/` → `src/pm_server/` にリネーム。

```bash
mv src/pm_agent src/pm_server
```

### タスク 2: pyproject.toml 更新

以下のフィールドを変更:

```toml
[project]
name = "pm-server"
description = "Project management MCP Server for Claude Code — track tasks, visualize progress, manage decisions"
authors = [
    { name = "Shinichi Nakazato" }
]
keywords = ["claude-code", "project-management", "mcp", "mcp-server"]

[project.scripts]
pm-server = "pm_server.__main__:cli"

[project.urls]
Homepage = "https://github.com/code-retriever/pm-server"
Repository = "https://github.com/code-retriever/pm-server"
Issues = "https://github.com/code-retriever/pm-server/issues"
```

旧 `pm-agent` エントリポイント、旧 URL は全て削除する。

### タスク 3: 全 import 文の一括置換

`src/pm_server/` 配下の全 .py ファイルで:
- `from pm_agent` → `from pm_server`
- `import pm_agent` → `import pm_server`
- `"pm-agent"` → `"pm-server"`（文字列リテラル内。ただし MCP ツール名 `pm_` プレフィックスは変更しない）

### タスク 4: server.py の FastMCP 名変更

```python
# 変更前
mcp = FastMCP("pm-agent")

# 変更後
mcp = FastMCP("pm-server")
```

### タスク 5: installer.py に migrate コマンド追加

`installer.py` に `migrate_from_pm_agent()` 関数を追加する:

```python
def migrate_from_pm_agent():
    """pm-agent から pm-server への移行。"""
    # 1. 旧 pm-agent の MCP 登録を解除
    try:
        subprocess.run(
            ["claude", "mcp", "remove", "pm-agent", "--scope", "user"],
            check=True, capture_output=True, text=True
        )
        print("✓ Old pm-agent MCP registration removed")
    except subprocess.CalledProcessError:
        print("  pm-agent was not registered (skipping)")
    except FileNotFoundError:
        print("  'claude' command not found (skipping removal)")

    # 2. 新 pm-server を登録
    install_mcp()

    # 3. registry チェック
    registry_path = Path.home() / ".pm" / "registry.yaml"
    if registry_path.exists():
        print(f"✓ Registry at {registry_path} is intact")
    else:
        print("⚠ Registry not found at ~/.pm/registry.yaml")

    # 4. CLAUDE.md 内の pm-agent 言及を警告
    if registry_path.exists():
        import yaml
        data = yaml.safe_load(registry_path.read_text()) or {}
        projects = data.get("projects", [])
        for proj in projects:
            proj_path = proj.get("path", "")
            claude_md = Path(proj_path) / "CLAUDE.md"
            if claude_md.exists():
                content = claude_md.read_text()
                if "pm-agent" in content or "pm_agent" in content:
                    print(f"⚠ {claude_md} contains 'pm-agent' references — please update manually")

    print("\n✓ Migration complete. Restart Claude Code to activate.")
```

また、`install_mcp()` と `uninstall_mcp()` 内の `"pm-agent"` 文字列も `"pm-server"` に変更すること。

### タスク 6: __main__.py に migrate コマンド追加

```python
@cli.command()
def migrate():
    """pm-agent からの移行。旧 MCP 登録を解除し pm-server として再登録。"""
    from .installer import migrate_from_pm_agent
    migrate_from_pm_agent()
```

### タスク 7: テストの import 修正

`tests/` 配下の全 .py ファイルで:
- `from pm_agent` → `from pm_server`
- `import pm_agent` → `import pm_server`
- `"pm-agent"` → `"pm-server"`（文字列リテラル内）

`test_installer.py` に migrate テストを追加:

```python
def test_migrate_from_pm_agent(tmp_path, monkeypatch):
    """migrate コマンドが旧 pm-agent を解除して pm-server を登録する。"""
    # subprocess.run をモックして claude mcp remove/add の呼び出しを検証
    calls = []
    def mock_run(cmd, **kwargs):
        calls.append(cmd)
        result = subprocess.CompletedProcess(cmd, 0, stdout="", stderr="")
        return result
    monkeypatch.setattr(subprocess, "run", mock_run)
    
    # registry を用意
    registry_dir = tmp_path / ".pm"
    registry_dir.mkdir()
    (registry_dir / "registry.yaml").write_text("projects: []")
    monkeypatch.setattr(Path, "home", lambda: tmp_path)
    
    from pm_server.installer import migrate_from_pm_agent
    migrate_from_pm_agent()
    
    # remove pm-agent が呼ばれたこと
    assert any("pm-agent" in str(c) for c in calls)
    # add pm-server が呼ばれたこと
    assert any("pm-server" in str(c) for c in calls)
```

### タスク 8: CLAUDE.md 更新

プロジェクトの `CLAUDE.md` 内の:
- `pm-agent` → `pm-server`
- `pm_agent` → `pm_server`

を全て置換。

### タスク 9: skill/SKILL.md 更新

`skill/SKILL.md` 内の:
- `pm-agent` → `pm-server`
- `pm_agent` → `pm_server`

を全て置換。

### タスク 10: docs/ 更新

`docs/` 配下の全 .md ファイルで `pm-agent` / `pm_agent` への言及を `pm-server` / `pm_server` に更新。
ただし `docs/design.md` は既に更新済みなので触らない。

### タスク 11: CHANGELOG.md 更新

CHANGELOG.md に v0.2.0 エントリを追加:

```markdown
## [0.2.0] - 2026-04-08

### Changed
- Package renamed from `pm-agent` to `pm-server` (PyPI name conflict with existing `PMAgent`)
- GitHub repository moved to `code-retriever/pm-server`
- Added `pm-server migrate` command for transitioning from pm-agent

### Added
- `README.ja.md` — Japanese README
- `migrate` CLI command for pm-agent → pm-server transition
```

### タスク 12: パッケージ再インストールと動作確認

```bash
pip uninstall pm-agent -y
pip install -e .
pm-server --version  # または pm-server --help
```

## ガード & 制約

- MCP ツール名（`pm_init`, `pm_status` 等の `pm_` プレフィックス）は変更しない
- `.pm/` ディレクトリ構造、YAML スキーマは一切変更しない
- `~/.pm/registry.yaml` のフォーマットは変更しない
- `docs/design.md` は既に v0.3.0 に更新済みなので上書きしない
- `README.md` と `README.ja.md` は既に更新済みなので上書きしない
- テストの registry 隔離（conftest.py の GLOBAL_PM_DIR モック）を壊さない

## テスト（検証手順）

1. `ruff check src/` がエラーなし
2. `ruff format --check src/` がエラーなし
3. `pytest` で全テストパス（115テスト以上）
4. `grep -r "pm_agent\|pm-agent" src/ tests/ skill/ CLAUDE.md` で残存がないこと
   ※ docs/design.md の変更履歴セクション、CHANGELOG.md の旧名言及は許容
5. `pm-server --help` が正常に表示
6. `pm-server serve` が起動する（Ctrl+C で終了）
7. `python -c "from pm_server import __version__; print(__version__)"` が動作
````

---
