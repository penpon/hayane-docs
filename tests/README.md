# pytest テストスイート

このディレクトリには、プロジェクトの自動テストが含まれています。

## 📋 目次

- [概要](#概要)
- [ディレクトリ構造](#ディレクトリ構造)
- [セットアップ](#セットアップ)
- [テストの実行](#テストの実行)
- [テスト戦略](#テスト戦略)
- [カバレッジ](#カバレッジ)
- [CI/CD](#cicd)
- [トラブルシューティング](#トラブルシューティング)

## 概要

このプロジェクトでは、**pytest** を使用した3層のテスト戦略を採用しています：

1. **ユニットテスト** (`tests/unit/`): 個々の関数やクラスをモックを使用して高速にテスト
2. **統合テスト** (`tests/integration/`): APIエンドポイントや複数コンポーネントの連携をテスト
3. **E2Eテスト** (`tests/e2e/`): 実際のDiscord通知を含む完全なフローをテスト（手動確認必要）

## ディレクトリ構造

```
tests/
├── conftest.py                      # 共通フィクスチャ定義
├── pytest.ini                       # pytest設定（ルートディレクトリ）
├── .env.test                        # テスト用環境変数
├── fixtures/                        # カスタムフィクスチャ
│   └── __init__.py
├── unit/                            # ユニットテスト
│   ├── test_api_handler.py         # API機能のテスト
│   ├── test_bot_core.py            # Bot機能のテスト
│   ├── test_config.py              # 設定管理のテスト
│   ├── test_discord_id_handler.py  # Discord IDハンドラのテスト
│   ├── test_dm_handler.py          # DM送信ハンドラのテスト
│   ├── test_notifications.py       # 通知ロジックのテスト
│   ├── test_sheets_handler.py      # Google Sheets APIハンドラのテスト
│   ├── test_stats_manager.py       # 統計管理機能のテスト
│   └── test_stats_manager_async.py # 統計管理非同期機能のテスト
├── integration/                     # 統合テスト
│   ├── test_api_endpoints.py       # APIエンドポイントのテスト
│   └── test_dm_flow.py             # DM送信フローのテスト
└── e2e/                             # E2Eテスト
    ├── test_discord_notifications.py
    └── test_manual_dm_reply.py
```

## セットアップ

### 1. 依存関係のインストール

```bash
# 通常の依存関係
pip install -r requirements.txt

# または Makefileを使用
make install
```

### 2. 環境変数の設定

`.env.test` ファイルがプロジェクトルートに作成されています。
必要に応じて値を調整してください：

```bash
# .env.test
TEST_MODE=true
API_KEY=test_api_key_for_pytest
DISCORD_TOKEN=test_discord_token_dummy
...
```

### 3. テスト環境の確認

```bash
# テスト環境が正しくセットアップされているか確認
make setup-test
```

## テストの実行

### 基本的な実行方法

```bash
# すべてのテスト（ユニット + 統合）
make test

# ユニットテストのみ
make test-unit

# 統合テストのみ
make test-integration

# E2Eテスト（Discord Bot起動必要）
make test-e2e
```

### pytest コマンド直接実行

```bash
# すべてのユニットテスト
pytest tests/unit/ -v

# 特定のテストファイル
pytest tests/unit/test_bot_core.py -v

# 特定のテストケース
pytest tests/unit/test_bot_core.py::TestSendDmById::test_send_dm_success -v

# マーカーでフィルタ
pytest -m unit                    # ユニットテストのみ
pytest -m integration             # 統合テストのみ
pytest -m "not e2e"               # E2E以外
```

### 高度な実行方法

```bash
# 並列実行（高速化）
make test-parallel
pytest tests/ -n auto

# 失敗したテストのみ再実行
make test-failed
pytest --lf

# ウォッチモード（ファイル変更時に自動再実行）
make test-watch

# 詳細な出力
pytest tests/unit/ -vv

# 特定のテストを名前で検索
pytest tests/ -k "test_send_dm" -v
```

## テスト戦略

### ユニットテスト (`tests/unit/`)

- **目的**: 個々の関数・クラスの動作を検証
- **特徴**:
  - モックを使用してDiscord API、Google Sheets APIを置き換え
  - 高速実行（数秒）
  - 外部依存なし
  - CI/CDで常に実行
- **カバレッジ目標**: 80%以上

**例**:
```python
@pytest.mark.asyncio
async def test_send_dm_success(mock_discord_user):
    result = await bot_core.send_dm_by_id(12345, "テスト")
    assert result["status"] == "success"
```

### 統合テスト (`tests/integration/`)

- **目的**: 複数コンポーネントの連携を検証
- **特徴**:
  - FlaskアプリのAPIエンドポイントを実際にテスト
  - モックを部分的に使用
  - CI/CDで実行（失敗しても続行）
- **カバレッジ目標**: 主要フローをカバー

**例**:
```python
def test_execute_bot_success(client, api_headers, sample_dm_data):
    response = client.post('/api/execute-bot', 
                          data=json.dumps(sample_dm_data),
                          headers=api_headers)
    assert response.status_code in [200, 202]
```

### E2Eテスト (`tests/e2e/`)

- **目的**: 実際のDiscord通知を含む完全なフローを検証
- **特徴**:
  - 実際のDiscord Botが必要
  - 実際の通知を送信
  - 手動確認が必要
  - CI/CDではスキップ
- **実行頻度**: リリース前、重要な変更時

**例**:
```python
@pytest.mark.e2e
@pytest.mark.skip(reason="実際のDM送信が必要")
def test_real_dm_sending(client, api_headers):
    # 実際のDMを送信
    ...
```

## カバレッジ

### カバレッジレポート生成

```bash
# HTMLレポート生成
make coverage

# レポート表示（ブラウザで開く）
make coverage-view

# pytestコマンド直接使用
pytest tests/unit/ tests/integration/ \
  --cov=modules \
  --cov=main \
  --cov-report=html \
  --cov-report=term
```

### カバレッジ目標

- **全体**: 70%以上
- **ユニットテスト**: 80%以上
- **統合テスト**: 主要APIエンドポイントをカバー

### カバレッジから除外

以下は自動的に除外されます：
- テストコード自体 (`tests/`)
- `if __name__ == "__main__":`
- 抽象メソッド

## CI/CD

### GitHub Actions

プルリクエストやプッシュ時に自動的にテストが実行されます：

- **ユニットテスト**: 常に実行（失敗時はビルド失敗）
- **統合テスト**: 実行（失敗しても続行）
- **E2Eテスト**: スキップ
- **Lintチェック**: flake8, black, isort
- **セキュリティチェック**: bandit, safety

設定ファイル: `.github/workflows/tests.yml`

### ローカルでCI/CD環境を再現

```bash
# CI環境変数を設定してテスト実行
CI=true make test

# Lintチェック
make lint

# コードフォーマット
make format
```

## pytest マーカー

テストは以下のマーカーで分類されています：

| マーカー | 説明 | 実行方法 |
|---------|------|---------|
| `unit` | ユニットテスト | `pytest -m unit` |
| `integration` | 統合テスト | `pytest -m integration` |
| `e2e` | E2Eテスト | `pytest -m e2e` |
| `slow` | 実行に時間がかかるテスト | `pytest -m "not slow"` |
| `requires_discord` | Discord Bot起動が必要 | `pytest -m "not requires_discord"` |
| `requires_api` | APIサーバー起動が必要 | `pytest -m "not requires_api"` |

## トラブルシューティング

### よくある問題

#### 1. `ModuleNotFoundError: No module named 'modules'`

**原因**: Pythonパスが正しく設定されていない

**解決策**:
```bash
# プロジェクトルートから実行
pytest tests/

# または PYTHONPATH を設定
export PYTHONPATH=$PWD
pytest tests/
```

#### 2. `fixture 'mock_discord_user' not found`

**原因**: `conftest.py` が読み込まれていない

**解決策**:
- `tests/conftest.py` が存在することを確認
- プロジェクトルートから実行

#### 3. テストが実行されない

**原因**: テストファイルの命名規則に従っていない

**解決策**:
- ファイル名を `test_*.py` に
- テスト関数を `test_*` に
- テストクラスを `Test*` に

#### 4. E2Eテストがスキップされる

**原因**: CI環境で実行している

**解決策**:
```bash
# ローカル環境で実行
unset CI
pytest tests/e2e/ -m e2e -s
```

#### 5. カバレッジが低い

**解決策**:
```bash
# カバレッジレポートを確認
make coverage
make coverage-view

# 未カバーの箇所を確認してテストを追加
```

## テスト実装の進行状況

1. **Phase 1** (完了): pytest基盤整備
2. **Phase 2** (完了): ユニットテスト実装
3. **Phase 3** (完了): 統合テスト実装
4. **Phase 4** (完了): E2Eテスト実装
5. **Phase 5** (完了): CI/CD設定
6. **Phase 6** (完了): ドキュメント整備

## 参考資料

- [pytest公式ドキュメント](https://docs.pytest.org/)
- [pytest-asyncio](https://pytest-asyncio.readthedocs.io/)
- [pytest-mock](https://pytest-mock.readthedocs.io/)
- [pytest-cov](https://pytest-cov.readthedocs.io/)

## 貢献

テストを追加・改善する場合：

1. 適切な層（unit/integration/e2e）にテストを配置
2. マーカーを適切に設定
3. `make test` で全テストが通ることを確認
4. カバレッジが下がらないことを確認
5. プルリクエストを作成

---

**質問・問題がある場合は、Issueを作成してください。**
