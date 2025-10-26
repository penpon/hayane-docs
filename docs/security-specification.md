# セキュリティ仕様書

Discord交流促進Botのセキュリティ機構と運用ガイドラインを定義します。

## 目次

1. [概要](#概要)
2. [環境分離セキュリティ](#環境分離セキュリティ)
3. [認証・認可](#認証認可)
4. [レート制限](#レート制限)
5. [アクセスログ](#アクセスログ)
6. [機密情報管理](#機密情報管理)
7. [通信セキュリティ](#通信セキュリティ)
8. [監査とモニタリング](#監査とモニタリング)
9. [インシデント対応](#インシデント対応)
10. [セキュリティチェックリスト](#セキュリティチェックリスト)

---

## 概要

本システムは、Discord APIとGoogle Sheets APIを利用する性質上、適切なセキュリティ対策が不可欠です。本ドキュメントでは、実装されているセキュリティ機構と運用上の注意事項を説明します。

### セキュリティ設計の基本方針

1. **多層防御（Defense in Depth）**: 複数のセキュリティ層を重ねて保護
2. **最小権限の原則**: 必要最小限の権限のみを付与
3. **環境分離**: 開発環境と本番環境を完全に分離
4. **監査可能性**: 全ての操作をログに記録
5. **自動検証**: 設定ミスを構造的に防止

---

## 環境分離セキュリティ

### 8層の防御機構

本システムでは、開発環境と本番環境を誤って混同することを**構造的に不可能**にするため、以下の8層の防御機構を実装しています。

#### 1. systemdサービスファイルでの分離

**目的**: サービスレベルで環境を明示的に指定

**実装**:
```ini
# hayane-dev.service
[Service]
Environment="APP_ENV=development"

# hayane-prod.service
[Service]
Environment="APP_ENV=production"
```

**効果**: サービス起動時に環境が自動的に決定され、手動での環境指定ミスを防止

---

#### 2. APP_ENV環境変数の必須化

**目的**: 環境が明示的に指定されていない場合は起動を拒否

**実装**:
```python
APP_ENV = os.getenv("APP_ENV")
if not APP_ENV:
    print("エラー: APP_ENV環境変数が設定されていません")
    sys.exit(1)
```

**効果**: 環境が不明な状態での起動を防止

---

#### 3. .envファイル存在チェック

**目的**: 環境指定なしの`.env`ファイルが存在する場合は起動を拒否

**実装**:
```python
if os.path.exists(".env"):
    print("エラー: .envファイルが存在します")
    print("環境別の設定ファイル(.env.development または .env.production)を使用してください")
    sys.exit(1)
```

**効果**: 誤って`.env`ファイルを作成した場合の事故を防止

---

#### 4. 環境変数ソースの整合性チェック

**目的**: シェル環境変数から継承された値を検出して起動を拒否

**実装**:
```python
# .envファイル読み込み前の環境変数を記録
pre_load_vars = dict(os.environ)

# .envファイル読み込み

# 読み込み後に差分をチェック
for key in REQUIRED_VARS:
    if key in pre_load_vars and os.getenv(key) == pre_load_vars[key]:
        print(f"警告: {key}がシェル環境変数から継承されています")
        sys.exit(1)
```

**効果**: 意図しない環境変数の混入を防止

---

#### 5. Flaskの自動読み込み無効化

**目的**: Flaskが自動的に`.env`ファイルを読み込むのを防止

**実装**:
```python
app = Flask(__name__)
app.config['ENV'] = APP_ENV
# Flaskの.env自動読み込みを無効化
```

**効果**: Flaskによる意図しない設定ファイルの読み込みを防止

---

#### 6. 必須環境変数の検証

**目的**: 必須の環境変数が全て設定されているかを確認

**実装**:
```python
REQUIRED_VARS = [
    "DISCORD_TOKEN",
    "GUILD_ID",
    "SPREADSHEET_ID",
    "SERVICE_ACCOUNT_FILE",
    "API_KEY",
    "STATS_API_TOKEN",
    # ... 他の必須変数
]

missing_vars = [var for var in REQUIRED_VARS if not os.getenv(var)]
if missing_vars:
    print(f"エラー: 以下の環境変数が設定されていません: {missing_vars}")
    sys.exit(1)
```

**効果**: 設定漏れによる起動失敗を事前に検出

---

#### 7. 環境識別子の検証

**目的**: `.env.{APP_ENV}`ファイル内の`ENVIRONMENT_NAME`が`APP_ENV`と一致するかを検証

**実装**:
```python
ENVIRONMENT_NAME = os.getenv("ENVIRONMENT_NAME")
if ENVIRONMENT_NAME != APP_ENV:
    print(f"エラー: 環境識別子が一致しません")
    print(f"  APP_ENV: {APP_ENV}")
    print(f"  ENVIRONMENT_NAME: {ENVIRONMENT_NAME}")
    sys.exit(1)
```

**効果**: 誤った設定ファイルの読み込みを検出

---

#### 8. 環境固有の設定値範囲チェック

**目的**: 推奨値から外れている場合は警告を表示

**実装**:
```python
# 開発環境の推奨ポート: 5001
# 本番環境の推奨ポート: 5000
if APP_ENV == "development" and PORT != 5001:
    print(f"警告: 開発環境の推奨ポートは5001です（現在: {PORT}）")

if APP_ENV == "production" and PORT != 5000:
    print(f"警告: 本番環境の推奨ポートは5000です（現在: {PORT}）")
```

**効果**: 設定ミスの早期発見

---

### 環境分離のベストプラクティス

#### ✅ 推奨される運用

1. **別々のDiscord Bot**: 開発用と本番用で異なるBotを使用
2. **別々のスプレッドシート**: データの混在を防止
3. **別々のサービスアカウント**: 権限の分離
4. **別々のAPI Key**: セキュリティトークンの分離
5. **別々の通知チャンネル**: 通知の混在を防止

#### ❌ 避けるべき運用

1. 同じDiscord Botを開発と本番で共用
2. 同じスプレッドシートを開発と本番で共用
3. `.env`ファイル（環境指定なし）の使用
4. 環境変数のシェルでの直接設定
5. 本番環境での`LOG_LEVEL=debug`

---

## 認証・認可

### API Key認証

#### 実装方式

**Bearer Token認証**を使用:
```
Authorization: Bearer {API_KEY}
```

#### API Keyの生成

強力なランダム文字列を使用してください。

**推奨方法**:
```bash
# 32バイトのランダムキー生成
openssl rand -base64 32
```

#### API Keyの管理

1. **環境変数で管理**: コードに直接記述しない
2. **定期的な変更**: 3-6ヶ月ごとに変更
3. **環境ごとに異なるキー**: 開発と本番で別のキーを使用
4. **アクセス制限**: 必要最小限のIPアドレスからのみアクセス許可（将来実装予定）

#### 認証失敗時の動作

**HTTPステータス**: `401 Unauthorized`

**レスポンス**:
```json
{
  "status": "error",
  "message": "認証に失敗しました",
  "error_code": "UNAUTHORIZED"
}
```

---

### Stats Token認証

統計情報取得専用の認証トークンです。

#### 目的

- API Keyとは別のトークンを使用することで、権限を分離
- 統計情報の閲覧権限のみを付与可能

#### 使用方法

API Key認証と同じくBearer Token認証を使用:
```
Authorization: Bearer {STATS_API_TOKEN}
```

---

## レート制限

### 目的

- **総当たり攻撃（Brute Force Attack）の防止**
- **サービス拒否攻撃（DoS）の緩和**
- **リソースの公平な分配**

### 実装

Flask-Limiterを使用した自動レート制限。

### 制限値一覧

| エンドポイント | 制限 | 理由 |
|---|---|---|
| `/` | 10回/分 | 情報取得のみ、頻繁なアクセス不要 |
| `/api/health` | 80回/分 | ヘルスチェック用、高頻度アクセス許可 |
| `/api/execute-bot` | 80回/分 | 主要機能、適度な制限 |
| `/api/stop-bot` | 10回/分 | 緊急停止用、頻繁な使用は想定外 |
| `/api/get-discord-id` | 30回/分 | 単発確認用、中程度の制限 |
| `/api/update-all-discord-ids` | 10回/分 | 一括更新、高負荷のため厳しく制限 |
| `/api/stats` | 80回/分 | 統計情報取得、高頻度アクセス許可 |
| `/api/stats/history` | 30回/分 | 履歴取得、中程度の制限 |
| `/api/test-notifications` | 10回/分 | テスト用、頻繁な使用は想定外 |
| `/api/test-dm-errors` | 10回/分 | テスト用、頻繁な使用は想定外 |

### レート制限超過時の動作

**HTTPステータス**: `429 Too Many Requests`

**レスポンス**:
```json
{
  "status": "error",
  "message": "レート制限を超過しました。しばらく待ってから再試行してください。",
  "error_code": "RATE_LIMIT_EXCEEDED",
  "retry_after": 60
}
```

**レスポンスヘッダー**:
```
X-RateLimit-Limit: 80
X-RateLimit-Remaining: 0
X-RateLimit-Reset: 1698134400
Retry-After: 60
```

---

## アクセスログ

### 目的

- **不正アクセスの検出**
- **将来的なIP Whitelist設定の準備**
- **アクセスパターンの分析**

### 実装

成功したAPIアクセスのIPアドレスを記録。

**ログファイル**: `api_access_success.log`

**ログフォーマット**:
```
2025-10-24 12:00:00 - 192.168.1.100 - /api/execute-bot - 200
2025-10-24 12:01:00 - 192.168.1.100 - /api/stats - 200
```

### 運用

1. **定期的な確認**: 週1回程度、ログを確認
2. **不審なIPの検出**: 想定外のIPアドレスからのアクセスをチェック
3. **アクセスパターンの分析**: 異常なアクセス頻度を検出

---

## 機密情報管理

### 管理対象

以下の機密情報は厳重に管理する必要があります。

| 情報 | 格納場所 | 管理方法 |
|---|---|---|
| Discord Bot Token | `.env.{environment}` | 環境変数、Git除外 |
| API Key | `.env.{environment}` | 環境変数、Git除外 |
| Stats API Token | `.env.{environment}` | 環境変数、Git除外 |
| Google Service Account Key | `settings_{env}.json` | JSONファイル、Git除外 |
| Spreadsheet ID | `.env.{environment}` | 環境変数、Git除外 |

### .gitignoreの設定

以下のファイルは**必ずGit管理から除外**してください。

```gitignore
# 環境変数ファイル
.env
.env.development
.env.production

# Google認証情報
settings.json
settings_dev.json
settings_prod.json

# ログファイル
*.log
dm_logs.json

# その他の機密情報
api_keys.txt
secrets/
```

### 機密情報の取り扱いルール

#### ✅ 推奨される取り扱い

1. **環境変数で管理**: コードに直接記述しない
2. **定期的な変更**: 3-6ヶ月ごとに変更
3. **最小権限**: 必要最小限の権限のみ付与
4. **暗号化**: 可能な限り暗号化して保存
5. **アクセス制限**: 必要な人員のみアクセス可能に

#### ❌ 避けるべき取り扱い

1. コードに直接記述
2. Gitにコミット
3. チャットやメールで平文送信
4. スクリーンショットに含める
5. 公開リポジトリにアップロード

---

## 監査とモニタリング

### ログ記録

#### アプリケーションログ

**ログレベル**:
- `DEBUG`: 開発環境のみ
- `INFO`: 本番環境推奨
- `WARNING`: 警告レベルのイベント
- `ERROR`: エラーレベルのイベント
- `CRITICAL`: 致命的なエラー

**ログファイル**:
- 開発環境: `dev_logs/`
- 本番環境: `prod_logs/`

#### DM監視ログ

**ファイル**: `dm_logs.json`

**内容**:
- 受信したDMの記録
- 転送先チャンネル
- タイムスタンプ

#### APIアクセスログ

**ファイル**: `api_access_success.log`

**内容**:
- アクセス元IPアドレス
- エンドポイント
- HTTPステータスコード
- タイムスタンプ

---

### モニタリング

#### ヘルスチェック

定期的に`/api/health`エンドポイントを監視:
```bash
curl http://localhost:5000/api/health
```

**期待されるレスポンス**:
```json
{
  "status": "ok",
  "timestamp": "2025-10-24T12:00:00",
  "bot_status": "ready"
}
```

#### 統計情報の監視

定期的に統計情報を確認し、異常値を検出:
```bash
curl -H "Authorization: Bearer {STATS_API_TOKEN}" \
  http://localhost:5000/api/stats
```

**監視項目**:
- 成功率の急激な低下
- 失敗数の急増
- 異常なアクセスパターン

---
