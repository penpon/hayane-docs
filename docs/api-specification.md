# API仕様書

Discord交流促進BotのAPI仕様を定義します。

## 目次

1. [概要](#概要)
2. [認証](#認証)
3. [エンドポイント一覧](#エンドポイント一覧)
4. [エンドポイント詳細](#エンドポイント詳細)
5. [エラーレスポンス](#エラーレスポンス)
6. [レート制限](#レート制限)

---

## 概要

### ベースURL

- **開発環境**: `http://vip-ip-address:5001`
- **本番環境**: `http://vip-ip-address:5000`

### データフォーマット

- リクエスト: `application/json`
- レスポンス: `application/json`
- 文字エンコーディング: `UTF-8`

---

## 認証

本APIは2種類の認証方式を使用します。

### API Key認証

大部分のエンドポイントで使用される認証方式です。

**ヘッダー形式**:
```
Authorization: Bearer {API_KEY}
```

**対象エンドポイント**:
- `/`
- `/api/execute-bot`
- `/api/stop-bot`
- `/api/get-discord-id`
- `/api/update-all-discord-ids`
- `/api/test-notifications`
- `/api/test-dm-errors`

### Stats Token認証

統計情報取得専用の認証方式です。

**ヘッダー形式**:
```
Authorization: Bearer {STATS_API_TOKEN}
```

**対象エンドポイント**:
- `/api/stats`
- `/api/stats/history`

### 認証不要

以下のエンドポイントは認証不要です。

- `/api/health`

---

## エンドポイント一覧

| エンドポイント | メソッド | 認証 | 機能 |
|---|---|---|---|
| `/` | GET | API Key | API情報表示 |
| `/api/health` | GET | 不要 | ヘルスチェック |
| `/api/execute-bot` | POST | API Key | DM送信実行 |
| `/api/stop-bot` | POST | API Key | 処理停止 |
| `/api/get-discord-id` | POST | API Key | Discord ID取得（単発） |
| `/api/update-all-discord-ids` | POST | API Key | Discord ID一括更新 |
| `/api/stats` | GET | Stats Token | 現在の統計情報取得 |
| `/api/stats/history` | GET | Stats Token | 過去統計取得 |
| `/api/test-notifications` | POST | API Key | テスト通知送信 |
| `/api/test-dm-errors` | POST | API Key | DM送信エラーテスト |

---

## エンドポイント詳細

### GET /

API情報を表示します。

**認証**: API Key必須

**レスポンス例**:
```json
{
  "name": "Discord交流促進Bot API Server",
  "version": "2.0.0",
  "endpoints": {
    "/": "API情報",
    "/api/health": "ヘルスチェック",
    "/api/execute-bot": "DM送信実行",
    "/api/stop-bot": "処理停止",
    "/api/get-discord-id": "Discord ID取得",
    "/api/update-all-discord-ids": "Discord ID一括更新",
    "/api/stats": "統計情報取得",
    "/api/stats/history": "過去統計取得"
  }
}
```

---

### GET /api/health

サーバーの稼働状態を確認します。

**認証**: 不要

**レスポンス例**:
```json
{
  "status": "ok",
  "timestamp": "2025-10-24T12:00:00",
  "bot_status": "ready"
}
```

**ステータスコード**:
- `200`: 正常稼働中
- `503`: サービス利用不可

---

### POST /api/execute-bot

Discord DMを送信します。

**認証**: API Key必須

**リクエストボディ**:
```json
{
  "introducer_id": "123456789012345678",
  "introducee_ids": ["234567890123456789", "345678901234567890"],
  "message": "送信するメッセージの本文",
  "row_number": 3
}
```

**パラメータ**:
| フィールド | 型 | 必須 | 説明 |
|---|---|---|---|
| `introducer_id` | string | ✓ | 紹介者のDiscord ID |
| `introducee_ids` | array | ✓ | 被紹介者のDiscord IDリスト |
| `message` | string | ✓ | 送信するメッセージ本文 |
| `row_number` | integer | ✓ | スプレッドシートの行番号（ログ用） |

**成功レスポンス**:
```json
{
  "status": "success",
  "message": "送信完了",
  "sent_count": 2,
  "row_number": 3
}
```

**エラーレスポンス例**:
```json
{
  "status": "error",
  "message": "ユーザーが見つかりません",
  "error_code": "USER_NOT_FOUND",
  "row_number": 3
}
```

**ステータスコード**:
- `200`: 送信成功
- `400`: リクエストパラメータ不正
- `401`: 認証失敗
- `404`: ユーザーが見つからない
- `429`: レート制限超過
- `500`: サーバーエラー

---

### POST /api/stop-bot

実行中の処理を停止します。

**認証**: API Key必須

**リクエストボディ**: なし

**レスポンス例**:
```json
{
  "status": "success",
  "message": "処理を停止しました"
}
```

**ステータスコード**:
- `200`: 停止成功
- `401`: 認証失敗

---

### POST /api/get-discord-id

Discord表示名からIDを取得します（単発確認用）。

**認証**: API Key必須

**リクエストボディ**:
```json
{
  "display_name": "Hayane"
}
```

**パラメータ**:
| フィールド | 型 | 必須 | 説明 |
|---|---|---|---|
| `display_name` | string | ✓ | Discord表示名 |

**成功レスポンス**:
```json
{
  "status": "success",
  "discord_id": "123456789012345678",
  "display_name": "Hayane"
}
```

**エラーレスポンス例**:
```json
{
  "status": "error",
  "message": "ユーザーが見つかりません",
  "display_name": "Unknown User"
}
```

**ステータスコード**:
- `200`: 取得成功
- `400`: リクエストパラメータ不正
- `401`: 認証失敗
- `404`: ユーザーが見つからない

---

### POST /api/update-all-discord-ids

「顧客管理」シートのデータを「bot参照用」シートにコピーし、全Discord IDを一括更新します。

**認証**: API Key必須

**リクエストボディ**: なし

**レスポンス例**:
```json
{
  "status": "success",
  "match_count": 15,
  "already_filled_count": 3,
  "no_match_count": 2,
  "total_rows": 20,
  "details": [
    {
      "row": 2,
      "display_name": "Hayane",
      "discord_id": "123456789012345678",
      "status": "updated"
    },
    {
      "row": 3,
      "display_name": "Unknown",
      "discord_id": null,
      "status": "not_found"
    }
  ]
}
```

**レスポンスフィールド**:
| フィールド | 型 | 説明 |
|---|---|---|
| `match_count` | integer | ID取得成功数 |
| `already_filled_count` | integer | 既にIDが入力済みの数 |
| `no_match_count` | integer | ユーザーが見つからなかった数 |
| `total_rows` | integer | 処理対象の総行数 |
| `details` | array | 各行の詳細情報 |

**ステータスコード**:
- `200`: 処理完了
- `401`: 認証失敗
- `500`: サーバーエラー

---

### GET /api/stats

現在の統計情報を取得します。

**認証**: Stats Token必須

**レスポンス例**:
```json
{
  "status": "success",
  "total_sent": 150,
  "success_count": 145,
  "failure_count": 5,
  "success_rate": 96.67,
  "timestamp": "2025-10-24T12:00:00",
  "uptime_seconds": 86400
}
```

**レスポンスフィールド**:
| フィールド | 型 | 説明 |
|---|---|---|
| `total_sent` | integer | 総送信数 |
| `success_count` | integer | 成功数 |
| `failure_count` | integer | 失敗数 |
| `success_rate` | float | 成功率（%） |
| `timestamp` | string | 取得日時（ISO 8601形式） |
| `uptime_seconds` | integer | サーバー稼働時間（秒） |

**ステータスコード**:
- `200`: 取得成功
- `401`: 認証失敗

---

### GET /api/stats/history

過去の統計情報を取得します。

**認証**: Stats Token必須

**クエリパラメータ**:
| パラメータ | 型 | 必須 | 説明 | 例 |
|---|---|---|---|---|
| `days` | integer | - | 過去N日間のデータを取得 | `?days=7` |
| `from` | string | - | 開始日付（YYYY-MM-DD） | `?from=2025-10-01` |
| `to` | string | - | 終了日付（YYYY-MM-DD） | `?to=2025-10-24` |

**注意**: `days`と`from`/`to`は同時に指定できません。

**レスポンス例**:
```json
{
  "status": "success",
  "history": [
    {
      "date": "2025-10-24",
      "sent": 10,
      "success": 9,
      "failure": 1,
      "success_rate": 90.0
    },
    {
      "date": "2025-10-23",
      "sent": 15,
      "success": 14,
      "failure": 1,
      "success_rate": 93.33
    }
  ],
  "total_records": 2,
  "period": {
    "from": "2025-10-23",
    "to": "2025-10-24"
  }
}
```

**ステータスコード**:
- `200`: 取得成功
- `400`: パラメータ不正
- `401`: 認証失敗

---

### POST /api/test-notifications

テスト通知を送信します（開発・テスト用）。

**認証**: API Key必須

**リクエストボディ**:
```json
{
  "type": "health_check"
}
```

**利用可能な通知タイプ**:
| タイプ | 説明 |
|---|---|
| `health_check` | ヘルスチェック通知 |
| `daily_summary` | 日次統計サマリー通知 |
| `external_api_health` | 外部API接続確認 |
| `external_api_health_failure` | 外部API接続エラーシミュレート |
| `stats_anomaly` | 統計異常値検出通知 |

**`external_api_health_failure`の追加パラメータ**:
```json
{
  "type": "external_api_health_failure",
  "failure_type": "discord_only"
}
```

**failure_typeの値**:
- `discord_only`: Discord APIのみエラー
- `sheets_only`: Google Sheets APIのみエラー
- `both`: 両方エラー

**レスポンス例**:
```json
{
  "status": "success",
  "message": "テスト通知を送信しました",
  "notification_type": "health_check"
}
```

**ステータスコード**:
- `200`: 送信成功
- `400`: 通知タイプ不正
- `401`: 認証失敗

---

### POST /api/test-dm-errors

DM送信エラー通知をテストします（開発・テスト用）。

**認証**: API Key必須

**リクエストボディ**:
```json
{
  "error_type": "user_dm_disabled",
  "user_id": "123456789012345678"
}
```

**利用可能なエラータイプ**:
| タイプ | 説明 |
|---|---|
| `user_dm_disabled` | ユーザーがDMを無効化している |
| `rate_limited` | Discord APIのレート制限 |
| `server_error` | Discord APIサーバーエラー |
| `network_error` | ネットワークエラー |
| `unexpected` | 予期しないエラー |
| `dm_transfer_error` | DM転送エラー |

**レスポンス例**:
```json
{
  "status": "success",
  "message": "テストエラー通知を送信しました",
  "error_type": "user_dm_disabled"
}
```

**ステータスコード**:
- `200`: 送信成功
- `400`: エラータイプ不正
- `401`: 認証失敗

---

## エラーレスポンス

### 共通エラーフォーマット

```json
{
  "status": "error",
  "message": "エラーの詳細メッセージ",
  "error_code": "ERROR_CODE",
  "timestamp": "2025-10-24T12:00:00"
}
```

### エラーコード一覧

| コード | HTTPステータス | 説明 |
|---|---|---|
| `UNAUTHORIZED` | 401 | 認証失敗 |
| `INVALID_REQUEST` | 400 | リクエストパラメータ不正 |
| `USER_NOT_FOUND` | 404 | ユーザーが見つからない |
| `RATE_LIMIT_EXCEEDED` | 429 | レート制限超過 |
| `INTERNAL_ERROR` | 500 | サーバー内部エラー |
| `BOT_NOT_READY` | 503 | Botが準備中 |
| `DM_DISABLED` | 403 | ユーザーがDMを無効化 |
| `SPREADSHEET_ERROR` | 500 | スプレッドシート操作エラー |

---

## レート制限

各エンドポイントには以下のレート制限が設定されています。

| エンドポイント | 制限 |
|---|---|
| `/` | 10回/分 |
| `/api/health` | 80回/分 |
| `/api/execute-bot` | 80回/分 |
| `/api/stop-bot` | 10回/分 |
| `/api/get-discord-id` | 30回/分 |
| `/api/update-all-discord-ids` | 10回/分 |
| `/api/stats` | 80回/分 |
| `/api/stats/history` | 30回/分 |
| `/api/test-notifications` | 10回/分 |
| `/api/test-dm-errors` | 10回/分 |

### レート制限超過時のレスポンス

```json
{
  "status": "error",
  "message": "レート制限を超過しました。しばらく待ってから再試行してください。",
  "error_code": "RATE_LIMIT_EXCEEDED",
  "retry_after": 60
}
```

**HTTPステータスコード**: `429 Too Many Requests`

**ヘッダー**:
```
X-RateLimit-Limit: 80
X-RateLimit-Remaining: 0
X-RateLimit-Reset: 1698134400
Retry-After: 60
```

---
