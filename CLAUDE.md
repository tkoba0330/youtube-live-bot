# CLAUDE.md — YouTube Live Bot

## プロジェクト概要

YouTube Liveの配信を視聴し、音声のリアルタイム文字起こしと画面のスナップショット解析を行い、配信内容を理解した上でLLMが適切なコメントをライブチャットに自動投稿するBotシステム。

### 主要機能

1. **YouTube Live視聴** — Bot用アカウントでライブ配信に接続・視聴
2. **リアルタイム音声文字起こし** — 配信音声をストリーミングで取得し、Speech-to-Textで文字起こし
3. **画面スナップショット** — 定期的（および手動指示による）に配信画面をキャプチャし画像解析
4. **配信状況の理解・要約** — 文字起こし＋画像解析の結果からLLMが配信内容を総合的に理解
5. **コメント自動投稿** — LLMがキャラクター設定に基づいたコメントを生成し、YouTube Live Chatに投稿
6. **マルチBot対応** — Botごとに異なるキャラクター（システムプロンプト）を設定可能

---

## 推奨システム構成

### アーキテクチャ概要

```
┌─────────────────────────────────────────────────────────┐
│                    YouTube Live Stream                    │
└──────────┬──────────────────────┬────────────────────────┘
           │                      │
     yt-dlp + ffmpeg         YouTube Data API v3
           │                      │
    ┌──────┴──────┐          ┌────┴─────┐
    │  Audio      │          │  Live    │
    │  Pipeline   │          │  Chat    │
    │             │          │  API     │
    │  ┌────────┐ │          │          │
    │  │Whisper │ │          │ list/    │
    │  │(STT)   │ │          │ insert   │
    │  └───┬────┘ │          └────┬─────┘
    │      │      │               │
    ├──────┤      │               │
    │  ┌────────┐ │               │
    │  │Frame   │ │               │
    │  │Capture │ │               │
    │  └───┬────┘ │               │
    └──────┼──────┘               │
           │                      │
    ┌──────┴──────────────────────┴──────┐
    │         Context Manager            │
    │  (文字起こし + 画像解析 の蓄積)      │
    └──────────────┬─────────────────────┘
                   │
    ┌──────────────┴─────────────────────┐
    │         LLM Comment Generator      │
    │  (キャラクター別システムプロンプト)    │
    └──────────────┬─────────────────────┘
                   │
            YouTube Live Chat
              (コメント投稿)
```

### コンポーネント構成

| コンポーネント | 役割 |
|---|---|
| **StreamCapture** | yt-dlp + ffmpegで配信の音声・映像を取得 |
| **AudioTranscriber** | faster-whisper / WhisperLiveKitで音声→テキスト変換 |
| **FrameCapture** | ffmpegで定期的/手動でスクリーンショット取得 |
| **ImageAnalyzer** | マルチモーダルLLM（Claude Vision等）で画像内容を解析 |
| **ContextManager** | 文字起こし・画像解析結果を時系列で蓄積・管理 |
| **CommentGenerator** | LLMがコンテキストからコメントを生成（キャラクター別） |
| **ChatPoster** | YouTube Data API v3でライブチャットにコメント投稿 |
| **BotOrchestrator** | 全コンポーネントの制御・スケジューリング |

---

## 推奨技術スタック

### 言語: Python 3.11+

Pythonを推奨する理由:
- faster-whisper, WhisperLiveKit等のSTTライブラリがPythonネイティブ
- ffmpeg-pythonによるメディア処理の充実したエコシステム
- Anthropic SDK, OpenAI SDK等のLLM APIクライアントが最も充実
- asyncioによる非同期処理でリアルタイムパイプラインを構築可能

### コア依存ライブラリ

```
# ストリーム取得
yt-dlp                  # YouTube配信のストリームURL取得・音声/映像ダウンロード
ffmpeg-python           # 音声抽出、フレームキャプチャ（ffmpegのPythonバインディング）

# 音声文字起こし
faster-whisper           # CTranslate2ベースの高速Whisper実装（ローカル実行）
# または
whisperlivekit           # リアルタイムストリーミング対応のWhisperラッパー

# 画像解析 / コメント生成
anthropic                # Claude API（Vision対応 / コメント生成）
# または
openai                   # GPT-4o（Vision対応 / コメント生成）

# YouTube API
google-api-python-client # YouTube Data API v3
google-auth-oauthlib     # OAuth 2.0 認証

# 非同期処理
asyncio                  # 非同期イベントループ（標準ライブラリ）
aiohttp                  # 非同期HTTPクライアント

# 設定・ユーティリティ
pydantic                 # 設定・データモデルのバリデーション
pydantic-settings        # 環境変数からの設定読み込み
python-dotenv            # .envファイル読み込み
structlog                # 構造化ログ
```

### 開発ツール

```
# パッケージ管理
uv                       # 高速なPythonパッケージマネージャ

# コード品質
ruff                     # Linter + Formatter（flake8 + black + isort の統合）
mypy                     # 型チェック
pytest                   # テストフレームワーク
pytest-asyncio           # 非同期テスト対応
```

---

## YouTube Data API の詳細

### 必要なOAuthスコープ

| スコープ | 用途 |
|---|---|
| `https://www.googleapis.com/auth/youtube` | ライブチャットの読み書き |
| `https://www.googleapis.com/auth/youtube.readonly` | ライブチャットの読み取りのみ |

### 主要エンドポイント

- **`liveChatMessages.list`** — チャットメッセージの取得（ポーリング）
- **`liveChatMessages.streamList`** — チャットメッセージのストリーミング取得（推奨・サーバープッシュ方式）
- **`liveChatMessages.insert`** — チャットへのメッセージ投稿
- **`liveBroadcasts.list`** — 配信情報の取得（liveChatId の特定に必要）

### API クォータ

- デフォルト: **10,000 units/日**（太平洋時間0時リセット）
- `list` 操作: 約 **1 unit**
- `insert` 操作: 約 **50 units**
- `streamList` はポーリング方式の `list` よりクォータ消費が少ない（推奨）
- クォータ不足の場合はGoogle API Consoleから増枠申請が可能

---

## 技術的な実現可能性と課題

### 実現可能（高確度）

| 項目 | 説明 |
|---|---|
| YouTube Live Chatへのコメント投稿 | YouTube Data API v3 `liveChatMessages.insert` で公式サポート |
| ライブ配信の音声取得 | yt-dlp + ffmpeg でストリームURLを取得しリアルタイムでパイプ処理可能 |
| 画面スナップショット取得 | ffmpeg で `-vf fps=1/N` により定期的なフレーム抽出が可能 |
| LLMによるコメント生成 | Claude / GPT-4o 等のAPIで、コンテキストに基づくテキスト生成は成熟した技術 |
| マルチモーダル画像解析 | Claude Vision / GPT-4o Vision で画像の内容理解が可能 |

### 技術的課題

#### 1. リアルタイム音声文字起こしの遅延（中程度）

- **課題**: Whisperはバッチ処理向けに設計されており、リアルタイムストリーミングでは遅延が発生する
- **対策**: WhisperLiveKit / SimulStreamingを使用。インテリジェントなバッファリングと逐次処理で3〜5秒程度の遅延に抑制可能
- **代替案**: Google Cloud Speech-to-Text Streaming API や Deepgram を使えば遅延を1秒以下に抑えられるが、コストが発生

#### 2. APIクォータ制限（中程度）

- **課題**: YouTube Data API のデフォルトクォータは10,000 units/日。高頻度の投稿はクォータを急速に消費
- **対策**:
  - `streamList` APIを使いポーリング回数を最小化
  - コメント投稿頻度を制限（例: 2〜5分に1回）
  - 複数Botの場合、それぞれ別のAPIプロジェクト/認証情報を使用

#### 3. YouTube利用規約・スパム判定（高リスク）

- **課題**: 自動コメント投稿はYouTubeのスパムフィルタに検出される可能性がある。YouTube API利用規約への準拠が必要
- **対策**:
  - コメント投稿頻度を人間的なペースに制限
  - 同一内容の繰り返し投稿を避ける
  - 自身が配信者であるチャンネルでの利用に限定する
  - APIの利用規約（YouTube API Services Terms of Service）を遵守

#### 4. 音声＋映像の同期処理（中程度）

- **課題**: 音声文字起こしと画面キャプチャのタイムスタンプを正確に同期する必要がある
- **対策**: ffmpegのパイプラインで音声と映像を同一ソースから分岐（`tee`/`-map`）し、タイムスタンプを共有

#### 5. 長時間配信でのメモリ・コンテキスト管理（中程度）

- **課題**: 数時間の配信では文字起こし・画像解析データが膨大になり、LLMのコンテキストウィンドウに収まらない
- **対策**:
  - スライディングウィンドウ方式でコンテキストを管理（直近N分のみ保持）
  - 定期的に要約を作成し、古いデータを要約で置き換え
  - 重要なイベント（盛り上がり等）のみを長期保持

#### 6. GPU要件（ローカルWhisper使用時）

- **課題**: faster-whisper をリアルタイムで実行するにはGPUが必要（CPU のみでは遅延が大きい）
- **対策**:
  - NVIDIA GPU + CUDA環境を用意（RTX 3060以上推奨）
  - GPUがない場合はクラウドSTTサービス（Deepgram, Google Cloud等）を利用
  - `large-v3` の代わりに `medium` や `small` モデルで妥協

#### 7. OAuth認証のトークン管理

- **課題**: YouTube APIのOAuthトークンは期限切れになるため、リフレッシュトークンの管理が必要
- **対策**: `google-auth-oauthlib` のトークンリフレッシュ機能を使い、自動更新を実装

---

## プロジェクト構成（予定）

```
youtube-live-bot/
├── CLAUDE.md                  # このファイル
├── README.md                  # プロジェクト説明
├── pyproject.toml             # プロジェクト設定・依存関係
├── .env.example               # 環境変数テンプレート
├── .gitignore
│
├── src/
│   └── youtube_live_bot/
│       ├── __init__.py
│       ├── main.py            # エントリーポイント
│       ├── config.py          # 設定管理（pydantic-settings）
│       │
│       ├── stream/            # ストリーム取得
│       │   ├── capture.py     # yt-dlp + ffmpeg によるストリーム取得
│       │   ├── audio.py       # 音声パイプライン
│       │   └── frame.py       # フレームキャプチャ
│       │
│       ├── transcribe/        # 音声文字起こし
│       │   ├── whisper.py     # faster-whisper ベースの文字起こし
│       │   └── base.py        # Transcriber基底クラス
│       │
│       ├── vision/            # 画像解析
│       │   ├── analyzer.py    # マルチモーダルLLMによる画像解析
│       │   └── base.py        # Analyzer基底クラス
│       │
│       ├── context/           # コンテキスト管理
│       │   └── manager.py     # 文字起こし・画像解析の蓄積・要約
│       │
│       ├── comment/           # コメント生成・投稿
│       │   ├── generator.py   # LLMによるコメント生成
│       │   └── poster.py      # YouTube API経由でチャット投稿
│       │
│       ├── bot/               # Bot管理
│       │   ├── orchestrator.py # 全体制御
│       │   └── character.py   # キャラクター定義・管理
│       │
│       └── youtube/           # YouTube API ラッパー
│           ├── auth.py        # OAuth認証
│           └── chat.py        # Live Chat API操作
│
├── characters/                # キャラクター設定ファイル
│   ├── example.yaml           # キャラクター設定テンプレート
│   └── ...
│
└── tests/
    ├── conftest.py
    ├── test_transcribe/
    ├── test_vision/
    ├── test_comment/
    └── test_context/
```

---

## 環境変数

```bash
# YouTube API
YOUTUBE_CLIENT_ID=              # Google Cloud Console OAuth Client ID
YOUTUBE_CLIENT_SECRET=          # Google Cloud Console OAuth Client Secret
YOUTUBE_REFRESH_TOKEN=          # OAuth Refresh Token

# LLM API（コメント生成・画像解析用）
ANTHROPIC_API_KEY=              # Claude API Key
# または
OPENAI_API_KEY=                 # OpenAI API Key

# Whisper設定
WHISPER_MODEL=large-v3          # faster-whisperモデル（tiny/base/small/medium/large-v3）
WHISPER_DEVICE=cuda             # cuda / cpu
WHISPER_LANGUAGE=ja             # 言語コード

# Bot設定
COMMENT_INTERVAL_SEC=180        # コメント投稿間隔（秒）
SNAPSHOT_INTERVAL_SEC=30        # スナップショット間隔（秒）
CONTEXT_WINDOW_MIN=10           # コンテキスト保持時間（分）
```

---

## 開発ワークフロー

```bash
# セットアップ
uv sync                          # 依存関係インストール

# コード品質
uv run ruff check .              # Lint
uv run ruff format .             # Format
uv run mypy src/                 # 型チェック

# テスト
uv run pytest                    # テスト実行
uv run pytest -x --tb=short      # 失敗時即停止

# 実行
uv run python -m youtube_live_bot --video-id <VIDEO_ID>
```

---

## コーディング規約

- Python 3.11+ の型ヒントを積極的に使用
- `async/await` を活用した非同期処理
- pydantic によるデータバリデーション
- ruff によるフォーマット・Lint統一
- 日本語コメントは最小限にし、コードの自己文書化を優先
- テストは pytest で記述、モック活用でAPI呼び出しをテスト

---

## 参考リンク

- [YouTube Live Streaming API — liveChatMessages](https://developers.google.com/youtube/v3/live/docs/liveChatMessages)
- [YouTube Data API — Quota Calculator](https://developers.google.com/youtube/v3/determine_quota_cost)
- [faster-whisper (GitHub)](https://github.com/SYSTRAN/faster-whisper)
- [WhisperLiveKit (PyPI)](https://pypi.org/project/whisperlivekit/)
- [yt-dlp (GitHub)](https://github.com/yt-dlp/yt-dlp)
- [Anthropic Claude API](https://docs.anthropic.com/)
