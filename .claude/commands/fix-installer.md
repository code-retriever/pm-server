installer.py を修正してください。

## 問題
現在の installer.py は `~/.claude/settings.json` に mcpServers を書いているが、
Claude Code はこのファイルからMCPを読まない。

## 正しい仕様（公式ドキュメント確認済み）

Claude Code の MCP 設定は以下の場所に保存される：
- **user scope**: `~/.claude.json` の `mcpServers` フィールド（全プロジェクト横断）
- **local scope**: `~/.claude.json` のプロジェクトパス配下
- **project scope**: プロジェクトルートの `.mcp.json`

pm-agent は全プロジェクトで使うので **user scope** が正解。

## 修正方法

`install_mcp()` で `claude mcp add` コマンドを subprocess で実行する：

```python
import subprocess
import shutil

def install_mcp():
    """Register pm-agent as a Claude Code MCP server (user scope)."""
    # pm-agent コマンドのフルパスを取得
    pm_agent_path = shutil.which("pm-agent")
    if pm_agent_path is None:
        print("✗ pm-agent command not found in PATH")
        return

    # claude コマンドの存在確認
    claude_path = shutil.which("claude")
    if claude_path is None:
        print("✗ claude command not found. Install Claude Code first.")
        return

    # 既に登録済みか確認
    result = subprocess.run(
        [claude_path, "mcp", "get", "pm-agent"],
        capture_output=True, text=True, timeout=10
    )
    if result.returncode == 0:
        print("✓ PM Agent is already registered in Claude Code")
        return

    # claude mcp add で user scope に登録
    result = subprocess.run(
        [claude_path, "mcp", "add", "--transport", "stdio",
         "--scope", "user", "pm-agent", "--",
         pm_agent_path, "serve"],
        capture_output=True, text=True, timeout=10
    )
    if result.returncode == 0:
        print("✓ PM Agent registered in Claude Code (user scope)")
        print("  Restart Claude Code to activate")
    else:
        print(f"✗ Failed to register: {result.stderr}")

def uninstall_mcp():
    """Remove pm-agent from Claude Code MCP servers."""
    claude_path = shutil.which("claude")
    if claude_path is None:
        print("✗ claude command not found")
        return

    result = subprocess.run(
        [claude_path, "mcp", "remove", "pm-agent"],
        capture_output=True, text=True, timeout=10
    )
    if result.returncode == 0:
        print("✓ PM Agent unregistered from Claude Code")
    else:
        print(f"PM Agent was not registered or removal failed: {result.stderr}")
```

## 旧コード（CLAUDE_SETTINGS_PATHS, find_claude_settings 等）は全削除

## テスト修正
test_installer.py も `subprocess.run` のモックに変更する。

## 完了後
- pytest tests/ -v で全テストパス確認
- ruff check + format
