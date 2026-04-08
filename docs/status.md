# PM Server — プロジェクト現況ドキュメント

**Date**: 2026-04-07
**Author**: Shinichi Nakazato / FLC design co., ltd.
**Version**: 0.1.0
**License**: MIT

---

## 1. 現在の状態

### サマリ

| 項目 | 状態 |
|---|---|
| コア実装 | ✅ 15 MCP ツール、全機能動作 |
| テスト | ✅ 115テスト全パス |
| コードレビュー | ✅ 全修正完了 |
| installer.py | ✅ `claude mcp add --scope user` 方式に修正済み |
| MCP 登録 | ✅ `~/.claude.json` に user scope で登録 |
| 実運用 | ✅ Rvim (38タスク完走) + 3プロジェクト |

### 実運用中プロジェクト

- rvim — Rust Vim Editor
- ZenResona
- SynapticKnowledge
- 切替ちゃん (KirikaeChan)

---

## 2. 既知の問題（優先順）

| # | 優先度 | 問題 | 対処 |
|---|---|---|---|
| 1 | 高 | テストが `~/.pm/registry.yaml` を汚染 | registry パスのモック化 |
| 2 | 中 | docs/design.md の Author 未修正 | `Shinichi Nakazato / FLC design co., ltd.` |
| 3 | 中 | Git コミット未整理 | `/git-organize` |
| 4 | 中 | CHANGELOG.md 未作成 | リリース前に作成 |
| 5 | 低 | GitHub リポジトリ未作成 | README整備後 |
| 6 | 低 | PyPI 未公開 | GitHub公開後 |

---

## 3. 次のアクション

### Phase 1: バグ修正
1. registry 汚染の修正（テストモック化）
2. pm_cleanup でテスト残骸除去
3. docs/design.md の Author 修正
4. Git コミット整理

### Phase 2: GitHub 公開準備
5. CHANGELOG.md
6. README にスクリーンショット
7. CI (pytest GitHub Actions)
8. リポジトリ作成 + push

### Phase 3: 将来
9. PyPI 公開
10. ダッシュボード改善
11. git hook 連携
