# PM Agent コードレビュー修正プロンプト

コードレビュー結果に基づき、以下の修正を順番に実施してください。

---

## 🔴 要修正（❌）— 必ず対応

### Fix 1: TEMPLATES_DIR のパッケージング問題（最優先）

**問題**: `dashboard.py:20` の `TEMPLATES_DIR` が `pip install` 後に不正パスを指す。テンプレートがパッケージに含まれない。

**修正手順**:
1. `templates/` を `src/pm_agent/templates/` に移動
2. `dashboard.py` の `TEMPLATES_DIR` を `importlib.resources` またはパッケージ相対パスに変更:
   ```python
   TEMPLATES_DIR = Path(__file__).parent / "templates"
   ```
3. `pyproject.toml` にテンプレート同梱設定を追加（Hatch の場合不要なことが多いが確認）
4. テスト: `pip install -e .` 後に `pm_dashboard` が HTML を生成できることを確認

### Fix 2: _load_yaml の YAMLError 未キャッチ

**問題**: `storage.py:42` で壊れた YAML を読むとクラッシュする。

**修正**:
```python
def _load_yaml(path: Path) -> dict | list | None:
    if not path.exists():
        return None
    text = path.read_text(encoding="utf-8")
    try:
        return yaml.safe_load(text)
    except yaml.YAMLError as e:
        raise PmAgentError(f"Failed to parse {path.name}: {e}") from e
```

**テスト追加**: `test_storage.py` に壊れた YAML を読むテストを追加。

### Fix 3: pm_discover の戻り値型注釈

**問題**: `server.py:447` の `-> list` が実際の `dict` 戻り値と不一致。

**修正**: 戻り値型を `-> dict` に修正。

### Fix 4: test_velocity.py の新規作成

**問題**: `velocity.py` にテストが全くない。

**テスト内容**:
- `calculate_velocity`: 0件 / 通常 / 1週間のみ の各ケース
- `detect_risks`: blocked タスク検知 / stale in-progress 検知 / estimate 超過検知
- トレンド判定: improving / declining / stable

---

## 🟡 警告（⚠️）— 推奨対応

### Fix 5: Cargo.toml パースエラーのキャッチ

`discovery.py:28-34` に try/except を追加:
```python
try:
    with open(cargo_toml, "rb") as f:
        cargo = tomllib.load(f)
    # ... existing code
except Exception:
    pass  # Cargo.toml が読めなくてもスキップ
```

### Fix 6: pm_next が blocked_by を考慮

`server.py` の pm_next 内で、`blocked_by` が空でないタスクも除外:
```python
if task.blocked_by:
    continue
```

### Fix 7: pm_init の冪等性

`server.py:84-102` で既存の project.yaml がある場合は上書きしない:
```python
project_yaml = pm_path / "project.yaml"
if not project_yaml.exists():
    save_project(pm_path, project)
```

### Fix 8: velocity.py のマジックナンバー定数化

```python
# velocity.py 先頭に定数定義
TREND_IMPROVING_THRESHOLD = 1.2
TREND_DECLINING_THRESHOLD = 0.8
STALE_DAYS_THRESHOLD = 7
ESTIMATE_OVERRUN_FACTOR = 1.5
```

### Fix 9: フェーズ期限超過の検知

`velocity.py` の `detect_risks` にフェーズ期限チェックを追加:
```python
# Phase overdue detection
for phase in project.phases:
    if phase.status != PhaseStatus.COMPLETED and phase.target_date:
        if phase.target_date < today:
            risks.append(...)
```

### Fix 10: フェーズ進捗計算の DRY 化

`server.py` と `dashboard.py` で重複しているフェーズ進捗計算・タスクステータス集計を、`utils.py` にヘルパー関数として抽出:
```python
# utils.py に追加
def calculate_phase_progress(tasks: list[Task], phase_id: str) -> dict:
    ...

def aggregate_task_status(tasks: list[Task]) -> dict:
    ...
```

### Fix 11: storage に add_risk / add_milestone 追加

`storage.py` に tasks/decisions と同様の個別追加関数を追加。

### Fix 12: 不足テストの追加

`test_server.py` に以下のテストを追加:
- pm_discover: 複数プロジェクトの検出
- pm_cleanup: 存在しないパスの検出
- pm_risks: リスク自動検知
- pm_velocity: ベロシティ計算
- pm_dashboard: HTML/text 生成

### Fix 13: CHANGELOG.md と GitHub Actions（後回し可）

- CHANGELOG.md を作成（0.1.0 の変更内容）
- .github/workflows/test.yml を作成（pytest CI）

---

## 実行順序

1. Fix 1（TEMPLATES_DIR）← PyPI 公開をブロックする致命的バグ
2. Fix 2（YAMLError）← 本番でユーザーが遭遇する確率が高い
3. Fix 3（型注釈）← 1行修正
4. Fix 4（test_velocity.py）← テストギャップ
5. Fix 5-9（警告の修正）
6. Fix 10-11（リファクタリング）
7. Fix 12（テスト追加）
8. Fix 13（CI/CD — 後回し可）

## テスト

修正完了後:
1. `pytest tests/ -v` で全テストパス（新規テスト含む）
2. `pip install -e .` 後に `pm-agent serve` が起動すること
3. `pm_dashboard` が pip install 環境で HTML 生成できること
4. 壊れた YAML を意図的に作成し、クラッシュせずエラーメッセージが出ること
