# CI/CDガイド

このドキュメントでは、hayaneプロジェクトのCI/CDパイプラインの詳細、デプロイ手順、トラブルシューティングについて説明します。

## 📋 目次

1. [CI/CD概要](#cicd概要)
2. [CI（継続的インテグレーション）](#ci継続的インテグレーション)
3. [CD（継続的デプロイ）](#cd継続的デプロイ)
4. [デプロイ手順](#デプロイ手順)
5. [トラブルシューティング](#トラブルシューティング)
6. [ベストプラクティス](#ベストプラクティス)

---

## CI/CD概要

### アーキテクチャ

```mermaid
graph TD
    A["開発フロー"]
    B["ローカル開発<br/>- コード修正<br/>- make test<br/>- git push"]
    C["CI（自動実行）<br/>- ユニットテスト<br/>- 統合テスト<br/>- Lintチェック<br/>- セキュリティチェック<br/>- カバレッジレポート"]
    D["CI成功を確認"]
    E["CD（手動実行）<br/>- GitHub Actionsで実行<br/>- デプロイ先を選択"]
    F["VPS上で自動実行<br/>- git pull<br/>- 依存関係更新<br/>- systemctl restart"]
    
    A --> B
    B --> C
    C --> D
    D --> E
    E --> F
```

### 使用技術

| 項目 | 技術 | 役割 |
|------|------|------|
| **CI/CDプラットフォーム** | GitHub Actions | ワークフローの自動実行 |
| **テストフレームワーク** | pytest | ユニット・統合・E2Eテスト |
| **Lintツール** | flake8, black, isort | コード品質チェック |
| **セキュリティチェック** | bandit, safety | 脆弱性スキャン |
| **カバレッジ** | pytest-cov, codecov | テストカバレッジ測定 |
| **デプロイ** | SSH + systemd | VPSへの自動デプロイ |

---

## CI（継続的インテグレーション）

### 概要

**自動実行**: `git push` 時に自動的に実行されます。

**対象ブランチ**:
- `main`
- `develop`
- Pull Request

### ワークフロー構成

`.github/workflows/tests.yml` で定義されています。

#### 1. テストジョブ（test）

```yaml
jobs:
  test:
    runs-on: ubuntu-latest
```

**実行内容**:

1. **チェックアウト**: リポジトリのコードを取得
2. **Python 3.12のセットアップ**: Python環境を構築
3. **uvのインストール**: 高速パッケージマネージャー
4. **依存関係のインストール**: requirements.txt + pytest関連
5. **ユニットテストの実行**: `tests/unit/` 配下のテスト
6. **統合テストの実行**: `tests/integration/` 配下のテスト
7. **カバレッジレポートのアップロード**: Codecovへアップロード
8. **テスト結果サマリーの生成**: GitHub Actions上で確認可能

**環境変数**:
```yaml
env:
  CI: true
  TEST_MODE: true
```

**カバレッジ目標**:
- ユニットテスト: 80%以上
- 統合テスト: 主要フローをカバー

#### 2. Lintジョブ（lint）

```yaml
jobs:
  lint:
    runs-on: ubuntu-latest
```

**実行内容**:

1. **Black**: コードフォーマットチェック
   ```bash
   black --check modules/ tests/ main.py
   ```

2. **isort**: インポート順序チェック
   ```bash
   isort --check-only modules/ tests/ main.py
   ```

3. **flake8**: コード品質チェック
   ```bash
   flake8 modules/ tests/ main.py --max-line-length=120
   ```

**注意**: Black と isort は警告のみ（`continue-on-error: true`）

#### 3. セキュリティジョブ（security）

```yaml
jobs:
  security:
    runs-on: ubuntu-latest
```

**実行内容**:

1. **bandit**: Pythonコードの脆弱性スキャン
   ```bash
   bandit -r modules/ -f json -o bandit-report.json
   ```

2. **safety**: 依存関係の脆弱性チェック
   ```bash
   safety check --json
   ```

**注意**: セキュリティ問題は警告として通知（`continue-on-error: true`）

### CI結果の確認

#### GitHubでの確認

1. **リポジトリのActionsタブにアクセス**
   ```
   https://github.com/your-username/hayane/actions
   ```

2. **最新のワークフロー実行を選択**

3. **各ジョブの結果を確認**
   - ✅ 緑色のチェックマーク: 成功
   - ❌ 赤色のバツマーク: 失敗
   - ⚠️ 黄色の警告マーク: 警告あり

#### テスト結果サマリー

GitHub Actionsの「Summary」タブで以下の情報を確認できます：

- **テスト環境**: Pythonバージョン、ブランチ
- **カバレッジ率（ファイル別）**: 各モジュールのカバレッジ率
- **総カバレッジ率**: プロジェクト全体のカバレッジ率

### ローカルでのCI再現

CI環境と同じテストをローカルで実行できます：

```bash
# 全テストを実行（CI環境と同じ）
make test

# カバレッジレポート生成
make coverage

# Lintチェック
black --check modules/ tests/ main.py
isort --check-only modules/ tests/ main.py
flake8 modules/ tests/ main.py --max-line-length=120

# セキュリティチェック
bandit -r modules/
safety check
```

---

## CD（継続的デプロイ）

### 概要

**手動実行**: GitHubのActionsタブから手動でデプロイを実行します。

**理由**:
- 本番環境への影響を最小化
- デプロイタイミングを運用者が制御
- 予期しない自動デプロイを防止

### ワークフロー構成

`.github/workflows/deploy-manual.yml` で定義されています。

#### デプロイジョブ（deploy）

```yaml
on:
  workflow_dispatch:
    inputs:
      environment:
        description: 'デプロイ先環境'
        required: true
        type: choice
        options:
          - production
          - development
```

**実行内容**:

1. **デプロイ環境の確認**: 選択された環境を表示
2. **VPSにSSH接続してデプロイ**: 以下の処理を自動実行
   - プロジェクトディレクトリに移動
   - 現在のブランチを確認
   - `git pull` で最新コードを取得
   - `requirements.txt` が変更されている場合、依存関係を再インストール
   - 環境に応じてサービスを再起動
     - `production`: `hayane-prod.service` を再起動
     - `development`: `hayane-dev.service` を再起動
   - サービスのステータスを確認

### 必要なGitHub Secrets

デプロイには以下のSecretsが必要です：

| Secret名 | 説明 | 例 |
|---------|------|-----|
| `VPS_HOST` | VPSのホスト名またはIPアドレス | `example.com` または `192.168.1.100` |
| `VPS_USER` | VPSのSSHユーザー名 | `ubuntu` |
| `SSH_PRIVATE_KEY` | SSH秘密鍵 | `-----BEGIN OPENSSH PRIVATE KEY-----...` |
| `VPS_PORT` | SSHポート（オプション、デフォルト: 22） | `22` |

#### Secretsの設定方法

1. **GitHubリポジトリの「Settings」タブにアクセス**
2. **「Secrets and variables」→「Actions」を選択**
3. **「New repository secret」をクリック**
4. **Secret名と値を入力して保存**

### デプロイ先環境

| 環境 | サービス名 | ポート | 用途 |
|------|-----------|--------|------|
| **production** | `hayane-prod.service` | 5000 | 本番環境 |
| **development** | `hayane-dev.service` | 5001 | 検証環境 |

---

## デプロイ手順

### 基本的なデプロイフロー

```
1. ローカルでコード修正
   ↓
2. make test（ローカルテスト）
   ↓
3. git add . && git commit -m "..."
   ↓
4. git push
   ↓
5. GitHub ActionsでCI実行（自動）
   ↓
6. CI成功を確認
   ↓
7. GitHubで「Deploy」ボタンをクリック（手動）
   ↓
8. デプロイ完了を確認
```

### 詳細手順

#### 1. ローカルでコード修正

```bash
# プロジェクトディレクトリに移動
cd /path/to/hayane

# 仮想環境を有効化
source .venv/bin/activate

# コードを修正
# ...
```

#### 2. ローカルテスト

```bash
# 全テストを実行
make test

# カバレッジレポート生成
make coverage

# Lintチェック
black modules/ tests/ main.py
isort modules/ tests/ main.py
flake8 modules/ tests/ main.py --max-line-length=120
```

#### 3. コミット＆プッシュ

```bash
# 変更をステージング
git add .

# コミット
git commit -m "feat: 新機能の追加"

# プッシュ
git push origin main
```

#### 4. CI実行を確認

1. **GitHubのActionsタブにアクセス**
   ```
   https://github.com/your-username/hayane/actions
   ```

2. **最新のワークフロー実行を選択**

3. **全ジョブが成功していることを確認**
   - ✅ test
   - ✅ lint
   - ✅ security

#### 5. デプロイ実行

1. **GitHubのActionsタブにアクセス**
   ```
   https://github.com/your-username/hayane/actions
   ```

2. **「Manual Deploy to VPS」を選択**

3. **「Run workflow」をクリック**

4. **デプロイ先を選択**
   - `production`: 本番環境（ポート5000）
   - `development`: 検証環境（ポート5001）

5. **「Run workflow」ボタンをクリックして実行**

#### 6. デプロイ完了を確認

1. **GitHub Actionsでデプロイジョブの完了を確認**

2. **VPS上でサービスのステータスを確認**
   ```bash
   # 本番環境
   sudo systemctl status hayane-prod

   # 検証環境
   sudo systemctl status hayane-dev
   ```

3. **ヘルスチェック**
   ```bash
   # 本番環境
   curl http://localhost:5000/api/health

   # 検証環境
   curl http://localhost:5001/api/health
   ```

   期待されるレスポンス:
   ```json
   {
     "status": "ok",
     "timestamp": "2025-10-24T12:00:00",
     "bot_status": "ready"
   }
   ```

### ロールバック手順

デプロイ後に問題が発生した場合、以下の手順でロールバックできます：

#### 方法1: 前のコミットにロールバック

```bash
# VPSにSSH接続
ssh user@your-vps

# プロジェクトディレクトリに移動
cd ~/workspace/hayane

# 前のコミットに戻す
git reset --hard HEAD~1

# サービスを再起動
sudo systemctl restart hayane-prod  # または hayane-dev
```

#### 方法2: 特定のコミットにロールバック

```bash
# VPSにSSH接続
ssh user@your-vps

# プロジェクトディレクトリに移動
cd ~/workspace/hayane

# コミット履歴を確認
git log --oneline

# 特定のコミットに戻す
git reset --hard <commit-hash>

# サービスを再起動
sudo systemctl restart hayane-prod  # または hayane-dev
```

---

## トラブルシューティング

### CI関連

#### テストが失敗する

**症状**: GitHub Actionsでテストジョブが失敗する

**原因**:
- テストコードのバグ
- 環境変数の不足
- 依存関係の問題

**対処法**:

1. **ローカルでテストを実行**
   ```bash
   make test
   ```

2. **エラーメッセージを確認**
   ```bash
   pytest tests/unit/ -v
   pytest tests/integration/ -v
   ```

3. **環境変数を確認**
   ```bash
   # .env.test が存在するか確認
   ls -la .env.test
   ```

4. **依存関係を再インストール**
   ```bash
   pip install -r requirements.txt
   ```

#### Lintチェックが失敗する

**症状**: GitHub Actionsでlintジョブが失敗する

**原因**:
- コードフォーマットの問題
- インポート順序の問題
- コード品質の問題

**対処法**:

1. **Blackでフォーマット**
   ```bash
   black modules/ tests/ main.py
   ```

2. **isortでインポート順序を修正**
   ```bash
   isort modules/ tests/ main.py
   ```

3. **flake8でコード品質を確認**
   ```bash
   flake8 modules/ tests/ main.py --max-line-length=120
   ```

#### セキュリティチェックで警告が出る

**症状**: GitHub Actionsでsecurityジョブが警告を出す

**原因**:
- コードに脆弱性がある
- 依存関係に脆弱性がある

**対処法**:

1. **banditレポートを確認**
   ```bash
   bandit -r modules/
   ```

2. **safetyで依存関係をチェック**
   ```bash
   safety check
   ```

3. **脆弱性を修正**
   - コードの修正
   - 依存関係のアップデート

### CD関連

#### デプロイが失敗する

**症状**: GitHub Actionsでデプロイジョブが失敗する

**原因**:
- SSH接続の問題
- VPS上のディスク容量不足
- サービスの起動失敗

**対処法**:

1. **SSH接続を確認**
   ```bash
   ssh user@your-vps
   ```

2. **VPS上のディスク容量を確認**
   ```bash
   df -h
   ```

3. **サービスのログを確認**
   ```bash
   sudo journalctl -u hayane-prod -n 50
   ```

4. **手動でサービスを再起動**
   ```bash
   sudo systemctl restart hayane-prod
   sudo systemctl status hayane-prod
   ```

#### サービスが起動しない

**症状**: デプロイ後にサービスが起動しない

**原因**:
- 環境変数の不足
- 依存関係の問題
- ポートの競合

**対処法**:

1. **サービスのステータスを確認**
   ```bash
   sudo systemctl status hayane-prod
   ```

2. **サービスのログを確認**
   ```bash
   sudo journalctl -u hayane-prod -n 50
   ```

3. **環境変数を確認**
   ```bash
   cat /path/to/.env.production
   ```

4. **ポートの使用状況を確認**
   ```bash
   sudo lsof -i :5000
   ```

5. **手動で起動してエラーを確認**
   ```bash
   cd ~/workspace/hayane
   source .venv/bin/activate
   APP_ENV=production python main.py
   ```

#### 依存関係のインストールが失敗する

**症状**: `requirements.txt` の更新時に依存関係のインストールが失敗する

**原因**:
- パッケージのバージョン競合
- ネットワークの問題
- ディスク容量不足

**対処法**:

1. **VPSにSSH接続**
   ```bash
   ssh user@your-vps
   ```

2. **手動で依存関係をインストール**
   ```bash
   cd ~/workspace/hayane
   source .venv/bin/activate
   pip install -r requirements.txt
   ```

3. **エラーメッセージを確認して対処**

---

## ベストプラクティス

### 開発フロー

1. **ローカルで必ずテストを実行**
   ```bash
   make test
   ```

2. **Lintチェックを実行**
   ```bash
   black modules/ tests/ main.py
   isort modules/ tests/ main.py
   flake8 modules/ tests/ main.py --max-line-length=120
   ```

3. **コミットメッセージは明確に**
   ```bash
   # 良い例
   git commit -m "feat: DM送信機能にリトライ機能を追加"
   git commit -m "fix: Discord ID取得時のエラーハンドリングを修正"
   
   # 悪い例
   git commit -m "update"
   git commit -m "fix bug"
   ```

4. **小さな変更を頻繁にコミット**
   - 大きな変更は分割する
   - 1つのコミットで1つの変更

### デプロイフロー

1. **まず検証環境でテスト**
   - `development` 環境でデプロイ
   - 動作確認
   - 問題なければ `production` 環境へ

2. **本番環境へのデプロイは慎重に**
   - CI が全て成功していることを確認
   - 検証環境で動作確認済みであることを確認
   - デプロイ後は必ずヘルスチェック

3. **デプロイ後の確認**
   ```bash
   # サービスのステータス確認
   sudo systemctl status hayane-prod
   
   # ログの確認
   sudo journalctl -u hayane-prod -n 50
   
   # ヘルスチェック
   curl http://localhost:5000/api/health
   ```

4. **問題が発生したらすぐにロールバック**
   - 問題を修正してから再デプロイ
   - ロールバック手順を事前に確認

### セキュリティ

1. **Secretsは絶対にコミットしない**
   - `.env.development`
   - `.env.production`
   - `settings_dev.json`
   - `settings_prod.json`

2. **定期的にセキュリティチェック**
   ```bash
   bandit -r modules/
   safety check
   ```

3. **依存関係を定期的に更新**
   ```bash
   pip list --outdated
   pip install --upgrade <package-name>
   ```

### モニタリング

1. **定期的にログを確認**
   ```bash
   sudo journalctl -u hayane-prod -n 100
   ```

2. **ディスク容量を監視**
   ```bash
   df -h
   ```

3. **サービスのステータスを監視**
   ```bash
   sudo systemctl status hayane-prod
   ```

---

## 関連ドキュメント

- **[README](../README.md)**: プロジェクト概要
- **[テストREADME](../tests/README.md)**: テストの詳細
- **[セットアップガイド](./setup-guide.md)**: 初期セットアップ手順
- **[運用ガイド](./operation-guide.md)**: 日常的な運用方法
- **[開発者ガイド](./developer-guide.md)**: 開発環境構築
