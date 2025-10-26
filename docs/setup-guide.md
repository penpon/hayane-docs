# セットアップガイド

Discord交流促進Botの詳細なセットアップ手順を説明します。

## 目次

1. [前提条件](#前提条件)
2. [Discord Botの作成](#discord-botの作成)
3. [Google Cloud Platformの設定](#google-cloud-platformの設定)
4. [スプレッドシートの準備](#スプレッドシートの準備)
5. [VPSの準備](#vpsの準備)
6. [プロジェクトのセットアップ](#プロジェクトのセットアップ)
7. [環境変数の設定](#環境変数の設定)
8. [Google Apps Scriptの設定](#google-apps-scriptの設定)
9. [systemdサービスの設定](#systemdサービスの設定)
10. [動作確認](#動作確認)
11. [トラブルシューティング](#トラブルシューティング)

---

## 前提条件

### 必要なアカウント

- [ ] Googleアカウント
- [ ] Discordアカウント
- [ ] VPS（Virtual Private Server）またはサーバー環境

### 必要なソフトウェア

- [ ] Python 3.9以上
- [ ] pip（Pythonパッケージマネージャー）
- [ ] git
- [ ] systemd（サービス管理用、Linux環境）

### 推奨スキル

- 基本的なコマンドライン操作
- 環境変数の理解
- JSON形式の理解

---

## Discord Botの作成

### 1. Discord Developer Portalへアクセス

1. [Discord Developer Portal](https://discord.com/developers/applications)にアクセス
2. Discordアカウントでログイン

### 2. 新しいアプリケーションの作成

1. 「New Application」ボタンをクリック
2. アプリケーション名を入力（例: `Hayane Bot Dev`、`Hayane Bot Prod`）
3. 「Create」をクリック

**重要**: 開発環境と本番環境で**別々のBot**を作成してください。

### 3. Botの作成

1. 左サイドバーから「Bot」タブを選択
2. 「Add Bot」ボタンをクリック
3. 確認ダイアログで「Yes, do it!」をクリック

### 4. Botトークンの取得

1. 「TOKEN」セクションで「Reset Token」をクリック
2. 表示されたトークンをコピー（後で`.env`ファイルに設定）
3. **重要**: このトークンは二度と表示されないため、安全な場所に保存

### 5. Privileged Gateway Intentsの有効化

以下のIntentsを有効にします：

1. 「Privileged Gateway Intents」セクションまでスクロール
2. 以下のトグルをONにする：
   - ✅ `PRESENCE INTENT`（オプション）
   - ✅ `SERVER MEMBERS INTENT`（必須）
   - ✅ `MESSAGE CONTENT INTENT`（必須）
3. 「Save Changes」をクリック

### 6. Bot権限の設定

1. 左サイドバーから「OAuth2」→「URL Generator」を選択
2. 「SCOPES」で以下を選択：
   - ✅ `bot`
3. 「BOT PERMISSIONS」で以下を選択：
   - ✅ `Send Messages`
   - ✅ `Read Messages/View Channels`
   - ✅ `Read Message History`
   - ✅ `Manage Messages`（オプション）

### 7. Botをサーバーに招待

1. 「GENERATED URL」セクションに表示されたURLをコピー
2. ブラウザの新しいタブでURLを開く
3. Botを招待するサーバーを選択
4. 「認証」をクリック

### 8. Guild IDの取得

1. Discordアプリで「ユーザー設定」→「詳細設定」→「開発者モード」をON
2. サーバーアイコンを右クリック→「IDをコピー」
3. コピーしたIDを保存（後で`.env`ファイルに設定）

### 9. チャンネルIDの取得

通知用チャンネルのIDを取得します：

1. 通知を送信したいチャンネルを右クリック
2. 「IDをコピー」を選択
3. 以下のチャンネルIDを取得：
   - 通常通知チャンネル（`NORMAL_NOTIFICATION_CHANNEL_ID`）
   - エラー通知チャンネル（`ERROR_NOTIFICATION_CHANNEL_ID`）
   - DM通知チャンネル（`DM_NOTIFICATION_CHANNEL_ID`）

---

## Google Cloud Platformの設定

### 1. Google Cloud Consoleへアクセス

1. [Google Cloud Console](https://console.cloud.google.com/)にアクセス
2. Googleアカウントでログイン

### 2. 新しいプロジェクトの作成

1. 画面上部のプロジェクト選択ドロップダウンをクリック
2. 「新しいプロジェクト」をクリック
3. プロジェクト名を入力（例: `hayane-bot-dev`、`hayane-bot-prod`）
4. 「作成」をクリック

**重要**: 開発環境と本番環境で**別々のプロジェクト**を作成してください。

### 3. Google Sheets APIの有効化

1. 左サイドバーから「APIとサービス」→「ライブラリ」を選択
2. 検索ボックスに「Google Sheets API」と入力
3. 「Google Sheets API」をクリック
4. 「有効にする」をクリック

### 4. サービスアカウントの作成

1. 左サイドバーから「APIとサービス」→「認証情報」を選択
2. 「認証情報を作成」→「サービスアカウント」をクリック
3. サービスアカウントの詳細を入力：
   - **名前**: `hayane-bot-service-account`
   - **説明**: `Discord交流促進Bot用サービスアカウント`
4. 「作成して続行」をクリック
5. ロールの選択（オプション、スキップ可能）
6. 「完了」をクリック

### 5. サービスアカウントキーの作成

1. 作成したサービスアカウントをクリック
2. 「キー」タブを選択
3. 「鍵を追加」→「新しい鍵を作成」をクリック
4. キーのタイプで「JSON」を選択
5. 「作成」をクリック
6. JSONファイルが自動的にダウンロードされます

### 6. サービスアカウントキーの配置

1. ダウンロードしたJSONファイルをリネーム：
   - 開発環境: `settings_dev.json`
   - 本番環境: `settings_prod.json`
2. プロジェクトのルートディレクトリに配置

### 7. サービスアカウントのメールアドレスを確認

JSONファイルを開き、`client_email`フィールドの値をコピー：
```json
{
  "client_email": "hayane-bot-service-account@project-id.iam.gserviceaccount.com",
  ...
}
```

このメールアドレスは後でスプレッドシートに共有する際に使用します。

---

## スプレッドシートの準備

### 1. サンプルスプレッドシートの作成

プロジェクトに含まれるスクリプトを使用してサンプルスプレッドシートを作成します。

```bash
# プロジェクトディレクトリに移動
cd /path/to/hayane

# 仮想環境を有効化（作成済みの場合）
source .venv/bin/activate

# 依存パッケージをインストール（初回のみ）
pip install -r requirements.txt

# サンプルスプレッドシート作成スクリプトを実行
python setup_existing_spreadsheet.py
```

### 2. スプレッドシートの確認

スクリプトが成功すると、以下のような出力が表示されます：

```
スプレッドシートが作成されました！
URL: https://docs.google.com/spreadsheets/d/SPREADSHEET_ID/edit
```

### 3. サービスアカウントに権限を付与

1. 出力されたURLをブラウザで開く
2. 右上の「共有」ボタンをクリック
3. サービスアカウントのメールアドレスを入力
4. 権限を「編集者」に設定
5. 「送信」をクリック

### 4. スプレッドシートIDの取得

URLから`SPREADSHEET_ID`部分をコピー：
```
https://docs.google.com/spreadsheets/d/SPREADSHEET_ID/edit
                                      ^^^^^^^^^^^^^^^^
```

このIDを後で`.env`ファイルに設定します。

### 5. シート構成の確認

作成されたスプレッドシートには以下のシートが含まれています：

- **顧客管理**: 顧客情報の管理
- **bot参照用**: Bot実行時に参照するデータ
- **bot管理用**: 送信管理とステータス記録

---

## VPSの準備

### 1. VPSの選択

以下のようなVPSプロバイダーを推奨します：

- AWS EC2
- Google Cloud Compute Engine
- DigitalOcean
- Vultr
- さくらのVPS

### 2. OSの選択

**推奨**: Ubuntu 22.04 LTS または Ubuntu 24.04 LTS

### 3. 基本的なサーバーセットアップ

```bash
# システムのアップデート
sudo apt update
sudo apt upgrade -y

# 必要なパッケージのインストール
sudo apt install -y python3 python3-pip python3-venv git

# Pythonバージョンの確認
python3 --version  # 3.9以上であることを確認
```

### 4. ファイアウォールの設定

```bash
# UFWのインストール（未インストールの場合）
sudo apt install -y ufw

# SSH接続を許可
sudo ufw allow 22/tcp

# アプリケーションポートを許可
sudo ufw allow 5000/tcp  # 本番環境
sudo ufw allow 5001/tcp  # 開発環境

# ファイアウォールを有効化
sudo ufw enable
```

---

## プロジェクトのセットアップ

### 1. プロジェクトのクローン

```bash
# ホームディレクトリに移動
cd ~

# プロジェクトをクローン
git clone https://github.com/your-username/hayane.git

# プロジェクトディレクトリに移動
cd hayane
```

### 2. 仮想環境の作成

```bash
# 仮想環境を作成
python3 -m venv .venv

# 仮想環境を有効化
source .venv/bin/activate
```

### 3. 依存パッケージのインストール

```bash
# requirements.txtから依存パッケージをインストール
pip install -r requirements.txt
```

### 4. ディレクトリ構造の確認

```bash
# プロジェクト構造を確認
tree -L 2
```

期待される出力：
```
hayane/
├── main.py
├── modules/
│   ├── __init__.py
│   ├── api_handler.py
│   ├── bot_core.py
│   ├── config.py
│   ├── discord_id_handler.py
│   ├── dm_handler.py
│   ├── sheets_handler.py
│   └── stats_manager.py
├── systemd/
│   ├── hayane-dev.service
│   ├── hayane-prod.service
│   └── install.sh
├── docs/
├── requirements.txt
├── .env.example
└── README.md
```

---

## 環境変数の設定

### 1. 環境別設定ファイルの作成

```bash
# .env.exampleをコピーして環境別ファイルを作成
cp .env.example .env.development
cp .env.example .env.production
```

**重要**: `.env`ファイル（環境指定なし）は作成しないでください。

### 2. 開発環境の設定（.env.development）

`.env.development`ファイルを編集：

```bash
nano .env.development
```

以下の内容を設定：

```env
# 環境識別子（必須、変更不可）
ENVIRONMENT_NAME=development

# Discord設定
DISCORD_TOKEN=your_development_discord_bot_token_here
GUILD_ID=your_development_guild_id_here

# Google Sheets設定
SPREADSHEET_ID=your_development_spreadsheet_id_here
SERVICE_ACCOUNT_FILE=settings_dev.json

# API認証
API_KEY=your_development_api_key_here
STATS_API_TOKEN=your_development_stats_api_token_here

# 通知チャンネル
NORMAL_NOTIFICATION_CHANNEL_ID=your_dev_normal_notification_channel_id_here
ERROR_NOTIFICATION_CHANNEL_ID=your_dev_error_notification_channel_id_here
DM_NOTIFICATION_CHANNEL_ID=your_dev_dm_notification_channel_id_here

# サーバー設定
HOST=0.0.0.0
PORT=5001

# ログ設定
LOG_LEVEL=debug
```

### 3. 本番環境の設定（.env.production）

`.env.production`ファイルを編集：

```bash
nano .env.production
```

以下の内容を設定：

```env
# 環境識別子（必須、変更不可）
ENVIRONMENT_NAME=production

# Discord設定
DISCORD_TOKEN=your_production_discord_bot_token_here
GUILD_ID=your_production_guild_id_here

# Google Sheets設定
SPREADSHEET_ID=your_production_spreadsheet_id_here
SERVICE_ACCOUNT_FILE=settings_prod.json

# API認証
API_KEY=your_production_api_key_here
STATS_API_TOKEN=your_production_stats_api_token_here

# 通知チャンネル
NORMAL_NOTIFICATION_CHANNEL_ID=your_prod_normal_notification_channel_id_here
ERROR_NOTIFICATION_CHANNEL_ID=your_prod_error_notification_channel_id_here
DM_NOTIFICATION_CHANNEL_ID=your_prod_dm_notification_channel_id_here

# サーバー設定
HOST=0.0.0.0
PORT=5000

# ログ設定
LOG_LEVEL=info
```

### 4. API Keyの生成

強力なランダムキーを生成します：

```bash
# API Keyの生成
openssl rand -base64 32

# Stats API Tokenの生成
openssl rand -base64 32
```

生成されたキーを`.env.development`と`.env.production`の該当箇所に設定してください。

### 5. サービスアカウントファイルの配置

1. Google Cloud Platformからダウンロードしたサービスアカウントキーをリネーム：
   ```bash
   mv ~/Downloads/your-project-xxxxx.json ~/hayane/settings_dev.json
   mv ~/Downloads/your-project-yyyyy.json ~/hayane/settings_prod.json
   ```

2. ファイルの配置を確認：
   ```bash
   ls -la | grep settings
   ```

   期待される出力：
   ```
   -rw------- 1 user user 2345 Oct 24 12:00 settings_dev.json
   -rw------- 1 user user 2345 Oct 24 12:00 settings_prod.json
   ```

---

## Google Apps Scriptの設定

### 1. スプレッドシートを開く

作成したスプレッドシートをブラウザで開きます。

### 2. Apps Scriptエディタを開く

1. メニューバーから「拡張機能」→「Apps Script」をクリック
2. 新しいタブでApps Scriptエディタが開きます

### 3. スクリプトをコピー

1. プロジェクトの`Code.gs`ファイルを開く
2. 内容を全てコピー
3. Apps Scriptエディタに貼り付け

### 4. スクリプトを保存

1. Ctrl+S（Mac: Cmd+S）で保存
2. プロジェクト名を入力（例: `Discord Bot Controller`）

### 5. スプレッドシートを再読み込み

1. スプレッドシートのタブに戻る
2. ページを再読み込み（F5またはCmd+R）
3. メニューバーに「Discord Bot」メニューが表示されることを確認

### 6. API Keyの設定

1. 「Discord Bot」メニュー→「API Key設定」をクリック
2. `.env.development`（または`.env.production`）に設定した`API_KEY`を入力
3. 「OK」をクリック

### 7. 初回実行時の認証

1. 「Discord Bot」メニューから任意の機能を実行
2. 「承認が必要です」ダイアログが表示される
3. 「権限を確認」をクリック
4. Googleアカウントを選択
5. 「詳細」→「安全ではないページに移動」をクリック
6. 「許可」をクリック

---

## systemdサービスの設定

### 1. サービスファイルのインストール

プロジェクトに含まれるインストールスクリプトを使用します：

```bash
cd ~/hayane/systemd
sudo bash install.sh
```

### 2. サービスファイルの確認

```bash
# 開発環境用サービス
sudo systemctl cat hayane-dev.service

# 本番環境用サービス
sudo systemctl cat hayane-prod.service
```

### 3. サービスの有効化

```bash
# 開発環境
sudo systemctl enable hayane-dev.service

# 本番環境
sudo systemctl enable hayane-prod.service
```

### 4. サービスの起動

```bash
# 開発環境を起動
sudo systemctl start hayane-dev.service

# 本番環境を起動
sudo systemctl start hayane-prod.service
```

### 5. サービスの状態確認

```bash
# 開発環境の状態確認
sudo systemctl status hayane-dev.service

# 本番環境の状態確認
sudo systemctl status hayane-prod.service
```

期待される出力：
```
● hayane-dev.service - Discord交流促進Bot (Development)
     Loaded: loaded (/etc/systemd/system/hayane-dev.service; enabled)
     Active: active (running) since Thu 2025-10-24 12:00:00 JST; 1min ago
   Main PID: 12345 (python)
      Tasks: 5 (limit: 4915)
     Memory: 50.0M
        CPU: 1.234s
     CGroup: /system.slice/hayane-dev.service
             └─12345 /home/user/hayane/.venv/bin/python /home/user/hayane/main.py
```

---

## 動作確認

### 1. ヘルスチェック

```bash
# 開発環境
curl http://localhost:5001/api/health

# 本番環境
curl http://localhost:5000/api/health
```

期待されるレスポンス：
```json
{
  "status": "ok",
  "timestamp": "2025-10-24T12:00:00",
  "bot_status": "ready"
}
```

### 2. Discord ID取得のテスト

```bash
# API Keyを環境変数に設定
export API_KEY="your_api_key_here"

# Discord ID取得
curl -X POST http://localhost:5001/api/get-discord-id \
  -H "Authorization: Bearer $API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"display_name": "YourDiscordName"}'
```

### 3. スプレッドシートからのテスト

1. スプレッドシートの「bot管理用」シートを開く
2. テストデータを1行入力
3. 「Discord Bot」メニュー→「送信実行」をクリック
4. 確認ダイアログで「はい」を選択
5. 送信結果を確認

### 4. ログの確認

```bash
# 開発環境のログ
sudo journalctl -u hayane-dev.service -f

# 本番環境のログ
sudo journalctl -u hayane-prod.service -f
```

---

## トラブルシューティング

### サービスが起動しない

**症状**: `sudo systemctl start hayane-dev.service`が失敗する

**確認事項**:

1. **環境変数の確認**
   ```bash
   # サービスファイルを確認
   sudo systemctl cat hayane-dev.service
   
   # APP_ENVが正しく設定されているか確認
   ```

2. **設定ファイルの存在確認**
   ```bash
   ls -la ~/hayane/.env.development
   ls -la ~/hayane/settings_dev.json
   ```

3. **ログの確認**
   ```bash
   sudo journalctl -u hayane-dev.service -n 50
   ```

### Discord Botが接続できない

**症状**: `bot_status: "not_ready"`

**確認事項**:

1. **Discord Tokenの確認**
   - `.env.development`の`DISCORD_TOKEN`が正しいか
   - Tokenが有効期限切れでないか

2. **Intentsの確認**
   - Discord Developer PortalでIntentsが有効になっているか

3. **ネットワーク接続の確認**
   ```bash
   ping discord.com
   ```

### Google Sheets APIエラー

**症状**: スプレッドシートへのアクセスが失敗する

**確認事項**:

1. **サービスアカウントキーの確認**
   ```bash
   cat ~/hayane/settings_dev.json | jq .client_email
   ```

2. **スプレッドシートの共有設定**
   - サービスアカウントのメールアドレスに編集権限があるか

3. **APIの有効化**
   - Google Cloud ConsoleでGoogle Sheets APIが有効になっているか

### ポートが既に使用されている

**症状**: `Address already in use`エラー

**解決方法**:

1. **使用中のプロセスを確認**
   ```bash
   sudo lsof -i :5001
   ```

2. **プロセスを停止**
   ```bash
   sudo kill -9 <PID>
   ```

3. **サービスを再起動**
   ```bash
   sudo systemctl restart hayane-dev.service
   ```

### 環境変数が読み込まれない

**症状**: 環境変数が`None`になる

**確認事項**:

1. **APP_ENVの設定**
   ```bash
   # サービスファイルで確認
   sudo systemctl cat hayane-dev.service | grep APP_ENV
   ```

2. **.envファイルの存在**
   ```bash
   # .envファイルが存在しないことを確認
   ls -la ~/hayane/.env
   # → "No such file or directory"が期待される
   ```

3. **環境別ファイルの存在**
   ```bash
   ls -la ~/hayane/.env.development
   ```

---

## 次のステップ

セットアップが完了したら、以下のドキュメントを参照してください：

- [運用ガイド](./operation-guide.md): 日常的な運用方法
- [API仕様書](./api-specification.md): API詳細仕様
- [セキュリティ仕様書](./security-specification.md): セキュリティ対策

---
