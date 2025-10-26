# 運用ガイド

Discord交流促進Botの日常的な運用方法とベストプラクティスを説明します。

## 目次

1. [日常的な運用](#日常的な運用)
2. [DM送信の実行](#dm送信の実行)
3. [Discord ID管理](#discord-id管理)
4. [サービスの管理](#サービスの管理)
5. [ログの確認](#ログの確認)
6. [統計情報の確認](#統計情報の確認)
7. [バックアップ](#バックアップ)

---

## 日常的な運用

### 運用フロー

```
1. スプレッドシートにデータ入力
   ↓
2. 重複チェックの確認
   ↓
3. Discord ID確認（必要に応じて）
   ↓
4. 送信実行
   ↓
5. 結果確認
   ↓
6. 統計情報の確認（定期的）
```

### 運用チェックリスト（日次）

- [ ] サービスの稼働状態を確認
- [ ] エラー通知チャンネルを確認
- [ ] 送信結果を確認
- [ ] 統計情報を確認（異常値がないか）

### 運用チェックリスト（週次）

- [ ] ログファイルのサイズを確認
- [ ] ディスク容量を確認
- [ ] アクセスログを確認（不審なIPがないか）
- [ ] バックアップの実施

### 運用チェックリスト（月次）

- [ ] 統計レポートの作成
- [ ] システムアップデートの確認
- [ ] セキュリティパッチの適用
- [ ] API Keyの変更（3-6ヶ月ごと）

---

## DM送信の実行

### 基本的な送信フロー

#### 1. スプレッドシートにデータを入力

1. スプレッドシートの「bot管理用」シートを開く
2. 以下の列にデータを入力：

| 列 | 項目 | 説明 | 例 |
|---|---|---|---|
| A | 紹介者Discord表示名 | 紹介する人の表示名 | `Hayane` |
| B | 紹介者Discord ID | 自動入力される | `123456789012345678` |
| C | 被紹介者1 Discord表示名 | 紹介される人1 | `Taro` |
| D | 被紹介者1 Discord ID | 自動入力される | `234567890123456789` |
| E | 被紹介者2 Discord表示名 | 紹介される人2（任意） | `Hanako` |
| F | 被紹介者2 Discord ID | 自動入力される | `345678901234567890` |
| G | メッセージ本文 | 送信するメッセージ | `こんにちは！...` |

#### 2. 重複チェックの確認

L列（重複チェック）を確認：
- ✅ **問題なし**: 送信可能
- ⚠️ **重複あり**: 過去に同じ組み合わせで送信済み

**重複の種類**:
- 順方向重複: 同じ組み合わせで送信済み
- 逆方向重複: 逆の組み合わせで送信済み

#### 3. Discord IDの確認

Discord IDが空欄の場合、以下の方法で取得：

**方法1: 単発確認**
1. 「Discord Bot」メニュー→「Discord ID確認（単発）」
2. Discord表示名を入力
3. IDが表示される

**方法2: 一括更新**
1. 「顧客管理」シートにデータを入力
2. 「Discord Bot」メニュー→「Discord ID一括更新」
3. 「bot参照用」シートに自動コピーされ、IDが更新される

#### 4. 送信実行

1. 「Discord Bot」メニュー→「送信実行」をクリック
2. 1行ずつ確認ダイアログが表示される
3. 内容を確認して「はい」または「いいえ」を選択
4. 送信が完了するまで待機

**確認ダイアログの内容**:
```
【送信確認】
行番号: 3
紹介者: Hayane (123456789012345678)
被紹介者: Taro (234567890123456789), Hanako (345678901234567890)

メッセージ:
こんにちは！...

この内容で送信しますか?
```

#### 5. 結果の確認

送信後、以下の列が自動更新されます：

| 列 | 項目 | 値 |
|---|---|---|
| I | 送信ステータス | `送信完了` / `エラー` / `スキップ` |
| J | エラー内容 | エラーの詳細（エラー時のみ） |
| K | 送信日時 | `2025-10-24 12:00:00` |

---

### 送信時の注意事項

#### ✅ 推奨される運用

1. **少量ずつ送信**: 一度に大量の送信を避ける
2. **確認を怠らない**: 送信前に必ず内容を確認
3. **重複チェック**: L列が「問題なし」であることを確認
4. **時間帯を考慮**: 深夜や早朝の送信を避ける
5. **エラー対応**: エラーが発生したら原因を確認してから再送信

#### ❌ 避けるべき運用

1. 確認せずに一括送信
2. 重複チェックを無視
3. エラーを放置
4. 短時間に大量送信
5. 不適切な時間帯の送信

---

### 緊急停止

送信中に問題が発生した場合、緊急停止が可能です。

**手順**:
1. 「Discord Bot」メニュー→「処理停止」をクリック
2. 確認ダイアログで「OK」をクリック
3. 現在の送信が完了後、処理が停止

**注意**: 既に送信されたメッセージは取り消せません。

---

## Discord ID管理

### Discord IDとは

Discord IDは、各ユーザーを一意に識別する18桁の数値です。

**特徴**:
- 表示名が変更されても変わらない
- サーバー間で共通
- 確実なユーザー特定が可能

### Discord IDの取得方法

#### 方法1: 単発確認

**用途**: 1人のIDを確認したい場合

**手順**:
1. 「Discord Bot」メニュー→「Discord ID確認（単発）」
2. Discord表示名を入力
3. IDが表示される
4. 必要に応じてスプレッドシートにコピー

#### 方法2: 一括更新

**用途**: 複数人のIDを一度に取得したい場合

**手順**:
1. 「顧客管理」シートにDiscord表示名を入力
2. 「Discord Bot」メニュー→「Discord ID一括更新」
3. 「bot参照用」シートに自動コピーされ、IDが更新される

**処理内容**:
- 「顧客管理」シートのデータを「bot参照用」シートにコピー
- サーバーメンバーリストから表示名に一致するユーザーを検索
- Discord IDを自動入力

**結果の確認**:
- ✅ **取得成功**: Discord IDが入力される
- ❌ **取得失敗**: 空欄のまま（ユーザーが見つからない）

### Discord IDが取得できない場合

**原因**:
1. 表示名が正確でない（スペース、大文字小文字など）
2. ユーザーがサーバーに参加していない
3. Botの権限が不足している

**対処法**:
1. 表示名を正確に確認（Discordアプリで確認）
2. ユーザーがサーバーに参加しているか確認
3. Botの権限を確認（SERVER MEMBERS INTENTが有効か）

---

## サービスの管理

### サービスの起動

```bash
# 開発環境を起動
sudo systemctl start hayane-dev.service

# 本番環境を起動
sudo systemctl start hayane-prod.service
```

### サービスの停止

```bash
# 開発環境を停止
sudo systemctl stop hayane-dev.service

# 本番環境を停止
sudo systemctl stop hayane-prod.service
```

### サービスの再起動

```bash
# 開発環境を再起動
sudo systemctl restart hayane-dev.service

# 本番環境を再起動
sudo systemctl restart hayane-prod.service
```

### サービスの状態確認

```bash
# 開発環境の状態確認
sudo systemctl status hayane-dev.service

# 本番環境の状態確認
sudo systemctl status hayane-prod.service
```

**期待される出力**:
```
● hayane-prod.service - Discord交流促進Bot (Production)
     Loaded: loaded (/etc/systemd/system/hayane-prod.service; enabled)
     Active: active (running) since Thu 2025-10-24 12:00:00 JST; 1h ago
   Main PID: 12345 (python)
      Tasks: 5 (limit: 4915)
     Memory: 50.0M
        CPU: 1.234s
```

### 自動起動の設定

```bash
# 開発環境の自動起動を有効化
sudo systemctl enable hayane-dev.service

# 本番環境の自動起動を有効化
sudo systemctl enable hayane-prod.service
```

### 自動起動の無効化

```bash
# 開発環境の自動起動を無効化
sudo systemctl disable hayane-dev.service

# 本番環境の自動起動を無効化
sudo systemctl disable hayane-prod.service
```

---

## ログの確認

### リアルタイムログの確認

```bash
# 開発環境のログをリアルタイム表示
sudo journalctl -u hayane-dev.service -f

# 本番環境のログをリアルタイム表示
sudo journalctl -u hayane-prod.service -f
```

### 過去のログの確認

```bash
# 最新100行を表示
sudo journalctl -u hayane-prod.service -n 100

# 特定の日付のログを表示
sudo journalctl -u hayane-prod.service --since "2025-10-24" --until "2025-10-25"

# エラーログのみ表示
sudo journalctl -u hayane-prod.service -p err
```

### ログレベル

| レベル | 説明 | 用途 |
|---|---|---|
| `DEBUG` | 詳細なデバッグ情報 | 開発環境 |
| `INFO` | 一般的な情報 | 本番環境 |
| `WARNING` | 警告 | 注意が必要な事象 |
| `ERROR` | エラー | エラーが発生 |
| `CRITICAL` | 致命的なエラー | システム停止レベル |

### ログファイルの場所

- **アプリケーションログ**: `prod_logs/` または `dev_logs/`
- **DM監視ログ**: `dm_logs.json`
- **APIアクセスログ**: `api_access_success.log`

### ログのローテーション

ログファイルが大きくなりすぎないよう、定期的にローテーションします。

**手動ローテーション**:
```bash
# ログファイルをアーカイブ
sudo journalctl --vacuum-time=30d  # 30日より古いログを削除
sudo journalctl --vacuum-size=1G   # 1GBを超える分を削除
```

---

## 統計情報の確認

### 現在の統計情報

```bash
# Stats API Tokenを環境変数に設定
export STATS_API_TOKEN="your_stats_api_token_here"

# 統計情報を取得
curl -H "Authorization: Bearer $STATS_API_TOKEN" \
  http://localhost:5000/api/stats | jq
```

**レスポンス例**:
```json
{
  "status": "success",
  "total_sent": 150,
  "success_count": 145,
  "failure_count": 5,
  "success_rate": 96.67,
  "timestamp": "2025-10-24T12:00:00"
}
```

### 過去の統計情報

```bash
# 過去7日間の統計
curl -H "Authorization: Bearer $STATS_API_TOKEN" \
  "http://localhost:5000/api/stats/history?days=7" | jq

# 特定期間の統計
curl -H "Authorization: Bearer $STATS_API_TOKEN" \
  "http://localhost:5000/api/stats/history?from=2025-10-01&to=2025-10-24" | jq
```

### 統計情報の解釈

**成功率の目安**:
- 95%以上: 良好
- 90-95%: 注意
- 90%未満: 要調査

**失敗の主な原因**:
1. ユーザーがDMを無効化している
2. ユーザーがサーバーから退出した
3. Discord APIのレート制限
4. ネットワークエラー

---

## バックアップ

### バックアップ対象

1. **スプレッドシート**: Googleドライブで自動バックアップ
2. **環境変数ファイル**: `.env.development`, `.env.production`
3. **サービスアカウントキー**: `settings_dev.json`, `settings_prod.json`
4. **ログファイル**: `dm_logs.json`, `api_access_success.log`

### バックアップ手順

#### 1. 環境変数ファイルのバックアップ

```bash
# バックアップディレクトリを作成
mkdir -p ~/hayane_backup/$(date +%Y%m%d)

# 環境変数ファイルをバックアップ
cp ~/hayane/.env.development ~/hayane_backup/$(date +%Y%m%d)/
cp ~/hayane/.env.production ~/hayane_backup/$(date +%Y%m%d)/
```

#### 2. サービスアカウントキーのバックアップ

```bash
# サービスアカウントキーをバックアップ
cp ~/hayane/settings_dev.json ~/hayane_backup/$(date +%Y%m%d)/
cp ~/hayane/settings_prod.json ~/hayane_backup/$(date +%Y%m%d)/
```

#### 3. ログファイルのバックアップ

```bash
# ログファイルをバックアップ
cp ~/hayane/dm_logs.json ~/hayane_backup/$(date +%Y%m%d)/
cp ~/hayane/api_access_success.log ~/hayane_backup/$(date +%Y%m%d)/
```

#### 4. バックアップの圧縮

```bash
# バックアップを圧縮
cd ~/hayane_backup
tar -czf hayane_backup_$(date +%Y%m%d).tar.gz $(date +%Y%m%d)/
```

### 自動バックアップの設定

cronを使用して定期的にバックアップを実行します。

```bash
# crontabを編集
crontab -e

# 以下を追加（毎日午前3時にバックアップ）
0 3 * * * /home/user/hayane/scripts/backup.sh
```

**backup.shの例**:
```bash
#!/bin/bash
BACKUP_DIR=~/hayane_backup/$(date +%Y%m%d)
mkdir -p $BACKUP_DIR
cp ~/hayane/.env.* $BACKUP_DIR/
cp ~/hayane/settings_*.json $BACKUP_DIR/
cp ~/hayane/*.log $BACKUP_DIR/
cp ~/hayane/dm_logs.json $BACKUP_DIR/
cd ~/hayane_backup
tar -czf hayane_backup_$(date +%Y%m%d).tar.gz $(date +%Y%m%d)/
rm -rf $(date +%Y%m%d)/
# 30日より古いバックアップを削除
find ~/hayane_backup -name "*.tar.gz" -mtime +30 -delete
```

---
