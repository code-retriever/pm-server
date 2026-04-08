# PM Agent コードレビュー

docs/design.md の設計仕様に対して、実装の整合性・品質・セキュリティを検証する。
以下のチェックリストを順番に実行し、各項目の結果を報告すること。

---

## 1. 設計適合性レビュー

docs/design.md を読み、各モジュールが設計どおりに実装されているか検証。

### 1.1 models.py
- [ ] 設計書 §3.3 のすべてのデータモデルが存在するか
- [ ] Enum 値が設計書の定義と一致するか（status, priority 等）
- [ ] フィールド名・型が YAML スキーマと一致するか
- [ ] Optional フィールドのデフォルト値が適切か
- [ ] バリデーションルール（ID形式、日付範囲）が実装されているか

### 1.2 storage.py
- [ ] 全 YAML ファイル種別（project, tasks, decisions, risks, milestones, daily）の CRUD が存在するか
- [ ] `safe_load` / `safe_dump` のみ使用しているか（`yaml.load` は禁止）
- [ ] ファイル未存在時のデフォルト生成が動作するか
- [ ] YAML 出力が human-readable か（`default_flow_style=False`, `allow_unicode=True`, `sort_keys=False`）
- [ ] registry.yaml の読み書きが正しいか

### 1.3 server.py
- [ ] 設計書 §4.1 の全17ツールが実装されているか（15でも可、差異を報告）
- [ ] 全ツールで `project_path` がオプショナルか
- [ ] `resolve_project_path` の探索ロジック（cwd → 親方向 .pm/ 探索）が正しいか
- [ ] タスクID自動採番が `{PREFIX}-{3桁連番}` 形式か
- [ ] pm_next の推薦ロジック（blocked除外、依存関係チェック）が正しいか

### 1.4 installer.py
- [ ] Claude Code settings.json の検出パスが正しいか
- [ ] 既存の settings.json のキーを壊さない実装になっているか
- [ ] 重複登録の防止チェックがあるか
- [ ] uninstall が正しく動作するか

### 1.5 discovery.py
- [ ] Cargo.toml / package.json / pyproject.toml / git / README からの推定が実装されているか
- [ ] discover_projects の再帰スキャンが正しいか
- [ ] エラーハンドリング（ファイル読み取り失敗、git コマンドタイムアウト等）

### 1.6 dashboard.py
- [ ] HTML ダッシュボード（単体ビュー + 横断ビュー）が生成されるか
- [ ] Chart.js が CDN から読み込まれるか
- [ ] ダークテーマが適用されているか
- [ ] テキストフォールバック（format="text"）が動作するか

### 1.7 velocity.py
- [ ] 週次ベロシティ計算が正しいか
- [ ] トレンド判定（上昇/下降/横ばい）ロジック
- [ ] リスク自動検知（期限超過、長期blocked）

### 1.8 __main__.py
- [ ] install / uninstall / serve / discover / status コマンドが存在するか
- [ ] `pm-agent serve` が FastMCP を stdio で起動するか

---

## 2. セキュリティレビュー

- [ ] `yaml.safe_load` のみ使用（`yaml.load`, `yaml.unsafe_load` は禁止）
- [ ] ファイルパスの正規化（`resolve()`）でパストラバーサルを防止しているか
- [ ] installer.py が JSON を部分更新（既存キー保護）しているか
- [ ] subprocess 呼び出し（git コマンド等）に `timeout` が設定されているか
- [ ] ユーザー入力が YAML インジェクションに対して安全か
- [ ] `shell=True` を使っていないか（subprocess）

---

## 3. エラーハンドリングレビュー

- [ ] .pm/ ディレクトリ未存在時のエラーメッセージが明確か
- [ ] 壊れた YAML ファイルの読み込み時にクラッシュしないか
- [ ] 存在しないタスクIDの更新時のエラー処理
- [ ] 権限不足でファイル書き込みできない場合のハンドリング
- [ ] registry.yaml 内の存在しないパスへのアクセス時
- [ ] MCP ツールが例外を投げずに dict/list でエラーを返すか

---

## 4. エッジケースレビュー

- [ ] 空のプロジェクト（タスク0件）で pm_status / pm_next / pm_dashboard が動作するか
- [ ] タスクID 999 を超えた場合の採番（RVIM-1000 等）
- [ ] 日本語を含むプロジェクト名・タスク名の YAML シリアライズ
- [ ] 非常に長い description や notes
- [ ] 同時に pm_init を2回実行した場合の冪等性
- [ ] registry.yaml に同じパスを2重登録しようとした場合
- [ ] .pm/ が存在するが project.yaml が壊れている場合

---

## 5. テストカバレッジレビュー

各テストファイルを読み、以下を確認：

- [ ] 正常系テストが全ツール/全関数に対して存在するか
- [ ] 異常系テスト（不正入力、ファイル未存在等）が存在するか
- [ ] フィクスチャ（conftest.py）が適切に共有されているか
- [ ] テストが tmp_path を使い、実ファイルシステムを汚さないか
- [ ] モック使用（installer.py のファイル操作等）が適切か
- [ ] テストが独立して実行可能か（テスト間の依存がないか）

---

## 6. コード品質レビュー

- [ ] 全 pub 関数に docstring があるか
- [ ] 型ヒントが全関数に記述されているか
- [ ] 未使用の import がないか
- [ ] マジックナンバーが定数化されているか
- [ ] DRY 原則（重複コードがないか）
- [ ] 関数の長さが適切か（50行超の関数がないか）
- [ ] 命名が CLAUDE.md の規約に従っているか

---

## 7. 出力形式

各セクションの結果を以下の形式で報告：

```
## セクション名

✅ チェック項目 — OK
⚠️ チェック項目 — 軽微な問題：説明
❌ チェック項目 — 要修正：説明 + 修正案

### 修正が必要な箇所

1. ファイル:行番号 — 問題の説明 — 修正コード
2. ...
```

最後にサマリーを出力：
- ✅ パス数
- ⚠️ 警告数
- ❌ 要修正数
- 総合評価（Ship it / Minor fixes / Major revision needed）
