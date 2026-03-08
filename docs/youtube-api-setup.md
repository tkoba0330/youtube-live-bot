# YouTube API セットアップガイド

YouTube Live Botを動作させるために必要な、YouTubeチャンネルの開設からAPIキーの取得、Botの接続確認までの手順を説明します。

---

## 目次

1. [Bot用Googleアカウントの準備](#1-bot用googleアカウントの準備)
2. [YouTubeチャンネルの開設](#2-youtubeチャンネルの開設)
3. [Google Cloudプロジェクトの作成](#3-google-cloudプロジェクトの作成)
4. [YouTube Data API v3の有効化](#4-youtube-data-api-v3の有効化)
5. [OAuth同意画面の設定](#5-oauth同意画面の設定)
6. [OAuth 2.0クライアントIDの作成](#6-oauth-20クライアントidの作成)
7. [リフレッシュトークンの取得](#7-リフレッシュトークンの取得)
8. [環境変数の設定](#8-環境変数の設定)
9. [接続確認](#9-接続確認)

---

## 1. Bot用Googleアカウントの準備

BotのコメントはBot用アカウントの名前で投稿されます。個人アカウントとは分けて、Bot専用のGoogleアカウントを作成してください。

1. https://accounts.google.com/signup にアクセス
2. Bot用の名前・メールアドレスを設定してアカウントを作成
   - 例: 表示名「配信アシスタントBot」、メール `youtube-bot-assistant@gmail.com`

> **注意**: YouTube APIはサービスアカウントに対応していません。必ず通常のGoogleアカウント（ユーザーアカウント）を使用してください。

---

## 2. YouTubeチャンネルの開設

Bot用GoogleアカウントでYouTubeチャンネルを作成します。

### 2-A. 個人チャンネルの作成（簡易）

1. Bot用Googleアカウントで https://www.youtube.com にログイン
2. 右上のアイコン → **「チャンネルを作成」** をクリック
3. チャンネル名（=Bot名）を入力して作成

### 2-B. ブランドアカウントで作成（推奨）

ブランドアカウントを使うと、複数人で管理でき、Googleアカウントの本名とは別の名前を設定できます。

1. https://www.youtube.com にログイン
2. 右上のアイコン → **「設定」** → **「チャンネルを追加または管理する」**
3. **「チャンネルを作成」** をクリック
4. Bot用のチャンネル名を入力して作成

> ブランドアカウントでは「管理者」を追加でき、チームで運用する場合に便利です。

---

## 3. Google Cloudプロジェクトの作成

YouTube Data APIを利用するにはGoogle Cloudプロジェクトが必要です。

1. https://console.cloud.google.com/ にアクセス（Bot用または管理用のGoogleアカウントでログイン）
2. 画面上部のプロジェクトセレクタ → **「新しいプロジェクト」** をクリック
3. 以下を入力:
   - **プロジェクト名**: `youtube-live-bot`（任意）
   - **組織**: なし、または所属組織を選択
4. **「作成」** をクリック

---

## 4. YouTube Data API v3の有効化

1. Google Cloud Console左メニューから **「APIとサービス」** → **「ライブラリ」** を開く
2. 検索ボックスで **「YouTube Data API v3」** を検索
3. **「YouTube Data API v3」** をクリック → **「有効にする」** をクリック

> 同様に **「YouTube Live Streaming API」** も検索して有効化してください（liveChatMessages等で必要）。

---

## 5. OAuth同意画面の設定

APIを利用するユーザーに表示される同意画面を設定します。

1. **「APIとサービス」** → **「OAuth同意画面」** を開く
2. **User Type** を選択:
   - **外部（External）**: Google Workspaceを使っていない場合はこちら
   - **内部（Internal）**: 組織内のユーザーのみに制限する場合（Google Workspace必須）
3. **「作成」** をクリック
4. 以下を入力:
   - **アプリ名**: `YouTube Live Bot`（任意）
   - **ユーザーサポートメール**: 自分のメールアドレス
   - **デベロッパーの連絡先情報**: 自分のメールアドレス
5. **「保存して次へ」** をクリック

### スコープの追加

6. **「スコープを追加または削除」** をクリック
7. 以下のスコープを追加:

| スコープ | 用途 |
|---|---|
| `https://www.googleapis.com/auth/youtube.force-ssl` | ライブチャットの読み書き（推奨） |
| `https://www.googleapis.com/auth/youtube.readonly` | 配信情報の取得 |

8. **「保存して次へ」** をクリック

### テストユーザーの追加

9. **「ADD USERS」** をクリック
10. **Bot用Googleアカウントのメールアドレス**を追加
11. **「保存して次へ」** → 概要を確認して **「ダッシュボードに戻る」**

> **重要**: 「テスト」ステータスのアプリでは、テストユーザーに追加したアカウントのみがOAuth認証を通過できます。テストユーザーは最大100人まで追加可能です。

### 公開ステータスとトークン有効期限

| ステータス | リフレッシュトークンの有効期限 | 用途 |
|---|---|---|
| **テスト** | **7日で失効**（自動的に無効化される） | 開発・動作確認のみ |
| **本番（未確認）** | 長期間有効（明示的な期限なし） | 社内利用・100人未満 |
| **本番（確認済み）** | 長期間有効（明示的な期限なし） | 一般公開 |

> **重要**: Botを継続的に運用する場合は、OAuth同意画面を **「本番に公開」** にしてください。テストモードのままだとリフレッシュトークンが7日ごとに失効し、手動での再認証が必要になります。
>
> 100人未満の利用であれば **未確認（unverified）のまま本番公開** が可能です。初回認証時に「このアプリはGoogleで確認されていません」という警告が表示されますが、クリックして進めれば問題ありません。Googleの審査プロセスは不要です。

---

## 6. OAuth 2.0クライアントIDの作成

1. **「APIとサービス」** → **「認証情報」** を開く
2. 上部の **「+ 認証情報を作成」** → **「OAuthクライアントID」** を選択
3. 以下を設定:
   - **アプリケーションの種類**: **デスクトップアプリ**
   - **名前**: `youtube-live-bot-client`（任意）
4. **「作成」** をクリック
5. 表示される **クライアントID** と **クライアントシークレット** を控えておく

> **JSONをダウンロード** ボタンから `client_secret_XXXXX.json` をダウンロードして、プロジェクトルートに `client_secret.json` として保存しておくと、後のトークン取得スクリプトで使えます。

---

## 7. リフレッシュトークンの取得

OAuth 2.0のリフレッシュトークンを取得するため、以下のPythonスクリプトを実行します。

### 前提

```bash
pip install google-api-python-client google-auth-oauthlib
```

### トークン取得スクリプト

プロジェクトルートに `scripts/get_token.py` を作成して実行します。

```python
"""YouTube API OAuth 2.0 トークン取得スクリプト"""

from google_auth_oauthlib.flow import InstalledAppFlow

SCOPES = [
    "https://www.googleapis.com/auth/youtube.force-ssl",
    "https://www.googleapis.com/auth/youtube.readonly",
]


def main():
    # client_secret.json は手順6でダウンロードしたファイル
    flow = InstalledAppFlow.from_client_secrets_file(
        "client_secret.json",
        scopes=SCOPES,
    )

    # ブラウザが開き、Googleアカウントでの認証を求められます
    # Bot用アカウントでログインしてください
    credentials = flow.run_local_server(
        port=8080,
        access_type="offline",
        prompt="consent",  # 必ずリフレッシュトークンを取得する
    )

    print("=" * 60)
    print("以下の値を .env ファイルに設定してください:")
    print("=" * 60)
    print(f"YOUTUBE_CLIENT_ID={credentials.client_id}")
    print(f"YOUTUBE_CLIENT_SECRET={credentials.client_secret}")
    print(f"YOUTUBE_REFRESH_TOKEN={credentials.refresh_token}")
    print("=" * 60)


if __name__ == "__main__":
    main()
```

### 実行手順

```bash
python scripts/get_token.py
```

1. ブラウザが自動的に開き、Googleの認証画面が表示される
2. **Bot用Googleアカウント**でログイン
3. 「このアプリはGoogleで確認されていません」と表示されたら → **「詳細」** → **「〇〇（安全でないページ）に移動」** をクリック
4. スコープの確認画面で **「続行」** をクリック
5. ターミナルに `YOUTUBE_CLIENT_ID`、`YOUTUBE_CLIENT_SECRET`、`YOUTUBE_REFRESH_TOKEN` が表示される

> **注意**: `prompt="consent"` を指定しないと、2回目以降の認証でリフレッシュトークンが返却されない場合があります。

### リフレッシュトークンが無効になるケース

以下のいずれかに該当すると、リフレッシュトークンが無効化されます。再度 `get_token.py` を実行して取得し直してください。

- OAuth同意画面が **「テスト」ステータス** のまま7日経過した
- ユーザーが https://myaccount.google.com/permissions でアプリのアクセスを **手動で取り消した**
- リフレッシュトークンが **6ヶ月間未使用** だった
- 同一ユーザー・同一OAuthクライアントで **100個以上のリフレッシュトークン** が発行された（最も古いものから失効）

---

## 8. 環境変数の設定

プロジェクトルートに `.env` ファイルを作成し、取得した値を設定します。

```bash
cp .env.example .env
```

```bash
# YouTube API
YOUTUBE_CLIENT_ID=xxxxxxxxxxxx.apps.googleusercontent.com
YOUTUBE_CLIENT_SECRET=GOCSPX-xxxxxxxxxxxxxxxx
YOUTUBE_REFRESH_TOKEN=1//xxxxxxxxxxxxxxxxxxxxxxxx
```

> `.env` ファイルは `.gitignore` に含めて、リポジトリにコミットしないでください。

---

## 9. 接続確認

設定が正しいか確認するためのスクリプトです。

### 確認スクリプト

プロジェクトルートに `scripts/verify_connection.py` を作成して実行します。

```python
"""YouTube API 接続確認スクリプト"""

import os

from dotenv import load_dotenv
from google.auth.transport.requests import Request
from google.oauth2.credentials import Credentials
from googleapiclient.discovery import build

load_dotenv()


def main():
    credentials = Credentials(
        token=None,
        refresh_token=os.environ["YOUTUBE_REFRESH_TOKEN"],
        client_id=os.environ["YOUTUBE_CLIENT_ID"],
        client_secret=os.environ["YOUTUBE_CLIENT_SECRET"],
        token_uri="https://oauth2.googleapis.com/token",
    )
    credentials.refresh(Request())

    youtube = build("youtube", "v3", credentials=credentials)

    # 1. 認証済みアカウントのチャンネル情報を取得
    response = youtube.channels().list(part="snippet", mine=True).execute()

    if not response.get("items"):
        print("エラー: チャンネルが見つかりません")
        print("Bot用アカウントでYouTubeチャンネルを作成してください")
        return

    channel = response["items"][0]
    print(f"チャンネル名: {channel['snippet']['title']}")
    print(f"チャンネルID: {channel['id']}")
    print()
    print("YouTube API接続に成功しました!")
    print()

    # 2. ライブ配信のチャットIDを取得（オプション）
    video_id = os.environ.get("TEST_VIDEO_ID")
    if video_id:
        video_response = (
            youtube.videos()
            .list(part="liveStreamingDetails", id=video_id)
            .execute()
        )
        if video_response.get("items"):
            details = video_response["items"][0].get("liveStreamingDetails", {})
            chat_id = details.get("activeLiveChatId")
            if chat_id:
                print(f"ライブチャットID: {chat_id}")
                print("ライブチャットへの接続も確認できました!")
            else:
                print(f"動画 {video_id} はライブ配信中ではないか、チャットが無効です")
        else:
            print(f"動画 {video_id} が見つかりません")
    else:
        print("ヒント: TEST_VIDEO_ID を.envに設定すると、ライブチャット接続も確認できます")


if __name__ == "__main__":
    main()
```

### 実行

```bash
pip install python-dotenv
python scripts/verify_connection.py
```

### 期待される出力

```
チャンネル名: 配信アシスタントBot
チャンネルID: UCxxxxxxxxxxxxxxxxxxxx

YouTube API接続に成功しました!

ヒント: TEST_VIDEO_ID を.envに設定すると、ライブチャット接続も確認できます
```

---

## トラブルシューティング

### `access_denied` エラー

- OAuth同意画面の **テストユーザー** にBot用アカウントのメールアドレスが追加されているか確認
- Bot用アカウントでログインしているか確認（個人アカウントではなく）

### `quotaExceeded` エラー

- デフォルトのクォータは **10,000 units/日**（太平洋時間0時リセット）
- `liveChatMessages.insert` は **50 units/回**
- クォータの使用状況は [Google Cloud Console の「割り当て」](https://console.cloud.google.com/apis/api/youtube.googleapis.com/quotas) で確認可能

### `liveChatNotFound` / `chatId` が取得できない

- 対象の動画がライブ配信中であることを確認
- 動画URLから `video_id` を正しく取得しているか確認（`https://www.youtube.com/watch?v=XXXXXXXXX` の `XXXXXXXXX` 部分）

### `NoLinkedYouTubeAccount` エラー

- サービスアカウントを使っていないか確認 → YouTube APIはサービスアカウント非対応
- Bot用アカウントにYouTubeチャンネルが紐づいているか確認

### リフレッシュトークンが取得できない

- `get_token.py` で `prompt="consent"` を指定しているか確認
- 一度 https://myaccount.google.com/permissions でアプリのアクセスを削除してから再実行

---

## APIクォータの目安

30分配信・90秒間隔でのコメント投稿を想定した場合:

| 操作 | 回数 | units/回 | 合計 |
|---|---|---|---|
| `liveChatMessages.insert`（コメント投稿） | 20回 | 50 | 1,000 |
| `liveChatMessages.list`（チャット取得） | ~360回 | 1 | 360 |
| `videos.list`（配信情報取得） | ~5回 | 1 | 5 |
| **合計** | | | **~1,365 units** |

デフォルトクォータ10,000 unitsに対して十分な余裕があります。

---

## 参考リンク

- [YouTube Data API v3 — Getting Started](https://developers.google.com/youtube/v3/getting-started)
- [YouTube Live Streaming API — liveChatMessages](https://developers.google.com/youtube/v3/live/docs/liveChatMessages)
- [OAuth 2.0 for Desktop Apps（YouTube Data API）](https://developers.google.com/youtube/v3/guides/auth/installed-apps)
- [OAuth同意画面の設定](https://developers.google.com/workspace/guides/configure-oauth-consent)
- [google-api-python-client — OAuth Guide](https://github.com/googleapis/google-api-python-client/blob/main/docs/oauth.md)
- [YouTube API Quota Calculator](https://developers.google.com/youtube/v3/determine_quota_cost)
