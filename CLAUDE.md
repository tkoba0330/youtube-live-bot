# CLAUDE.md — YouTube Live Bot

## プロジェクト概要

YouTube Liveの配信を視聴し、音声のリアルタイム文字起こしと画面のスナップショット解析を行い、配信内容を理解した上でLLMが適切なコメントをライブチャットに自動投稿するBotシステム。

### 利用シーン

- **社内向けYouTube Live配信**の対話補助Bot
- 配信者（自分）の話に対して、Botがコメントで**視聴者の理解を補助**する（要点整理、補足説明、質問の代弁など）
- 配信者が**脱線しすぎないよう制御**する役割（話題の逸脱を指摘、本題への誘導）
- 投稿頻度: **30分の配信で1〜3分に1回**（配信あたり10〜30コメント程度）

### 主要機能

1. **YouTube Live視聴** — Bot用アカウントでライブ配信に接続・視聴
2. **リアルタイム音声文字起こし** — 配信音声をストリーミングで取得し、Speech-to-Textで文字起こし
3. **画面スナップショット** — 定期的（および手動指示による）に配信画面をキャプチャし画像解析
4. **配信状況の理解・要約** — 文字起こし＋画像解析の結果からLLMが配信内容を総合的に理解
5. **コメント自動投稿** — LLMがキャラクター設定に基づいたコメントを生成し、YouTube Live Chatに投稿（視聴者補助・配信者への軌道修正）
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
                         # ※ yt-dlpはJSランタイム(deno/Node.js)が必要
ffmpeg-python           # 音声抽出、フレームキャプチャ（ffmpegのPythonバインディング）

# 音声文字起こし
faster-whisper           # CTranslate2ベースの高速Whisper実装（ローカル実行）
# または
whisperlivekit           # リアルタイムストリーミング対応のWhisperラッパー

# 画像解析 / コメント生成
anthropic                # Claude API（Vision対応 / コメント生成）
# または
openai                   # GPT-5.x（Vision対応 / コメント生成）

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
| `https://www.googleapis.com/auth/youtube.force-ssl` | ライブチャットの読み書き（推奨） |
| `https://www.googleapis.com/auth/youtube` | ライブチャットの読み書き（広範なスコープ） |
| `https://www.googleapis.com/auth/youtube.readonly` | ライブチャットの読み取りのみ |

> **注意**: サービスアカウントはYouTube Live Streaming APIでは使用不可。標準のOAuth 2.0ユーザー同意フローが必須。

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
- 参考: 5秒間隔でポーリング×1時間 ≈ 720 units、コメント投稿1回 = 50 units

### liveChatIdの取得フロー

1. `videos.list` で対象動画の `liveStreamingDetails.activeLiveChatId` を取得
2. 取得した `liveChatId` を使って `liveChatMessages.list` / `insert` を呼び出す

---

## STTサービス比較

| サービス | リアルタイム対応 | 遅延 | 料金（目安） | 特徴 |
|---|---|---|---|---|
| **Deepgram Nova-3** | WebSocket | <300ms | $0.26/hr | 最低遅延、ストリーミング特化 |
| **AssemblyAI** | WebSocket | <300ms | $0.37/hr | 高精度、テキスト整形が優秀 |
| **OpenAI Whisper API** | バッチのみ | 500ms+ | $0.36/hr | ストリーミング非対応 |
| **faster-whisper（ローカル）** | チャンク処理 | 1-5s | 無料（GPU必要） | セルフホスト、完全制御 |
| **Google Cloud STT** | gRPC | 中程度 | $1.00/hr | 多言語対応（100+） |

> **推奨**: コスト重視ならfaster-whisper（GPU必須）、低遅延重視ならDeepgram Nova-3、テキスト品質重視ならAssemblyAI

---

## マルチモーダルLLM比較（画像解析・コメント生成）（2026年3月時点）

### Anthropic Claude

| モデル | Vision | 入力/MTok | 出力/MTok | 特徴 |
|---|---|---|---|---|
| **Claude Opus 4.6** | Yes | $5 | $25 | 最高精度の推論・コード生成 |
| **Claude Sonnet 4.6** | Yes | $3 | $15 | 品質・コスト・速度のバランス最良 |
| **Claude Haiku 4.5** | Yes | $1 | $5 | 高速・低コスト、軽量タスク向け |

- Batch API: 50%割引、Prompt Caching: キャッシュ読み取り10%コスト
- 200K超のロングコンテキストは料金が上がる（Sonnet 4.6: $6/$22.50）

### OpenAI

| モデル | Vision | 入力/MTok | 出力/MTok | 特徴 |
|---|---|---|---|---|
| **GPT-5.4** | Yes | $2.50 | $15 | 最新フラッグシップ（2026年3月）、1Mコンテキスト |
| **GPT-5.2** | Yes | $1.75 | $14 | ScreenSpot-Pro 86.3%、優れた画像理解 |
| **GPT-5** | Yes | $1.25 | $10 | 安定した汎用マルチモーダルモデル |
| **GPT-5 Mini** | Yes | $0.25 | $2 | 高速・低コスト |
| **GPT-5 Nano** | Yes | $0.05 | $0.40 | 最安クラス、要約・分類向け |
| **GPT-4.1** | Yes | $2 | $8 | 旧世代、まだ利用可能 |

- Batch API: 50%割引、キャッシュ入力: 50-90%割引
- o系推論モデル（o3: $2/$8、o4-mini: $1.10/$4.40）もVision対応だがreasoning tokens別途消費

### Google Gemini

| モデル | Vision | 入力/MTok | 出力/MTok | 特徴 |
|---|---|---|---|---|
| **Gemini 3.1 Pro Preview** | Yes | $2 | $12 | 最新Pro（200K超: $4/$18） |
| **Gemini 3 Flash Preview** | Yes | $0.50 | $3 | MMMU Pro 79%、Vision最高クラスのコスパ |
| **Gemini 2.5 Pro** | Yes | $1.25 | $10 | 安定版、1Mコンテキスト |
| **Gemini 2.5 Flash** | Yes | $0.30 | $2.50 | 低コスト・高速、音声入力対応 |
| **Gemini 2.5 Flash-Lite** | Yes | $0.10 | $0.40 | 最安クラス |

- 画像入力: 1画像 ≈ 258トークン、音声・動画もネイティブ対応
- 無料枠あり（1日1,000リクエストまで）
- Gemini 3 Pro Preview は 2026/3/9 で非推奨 → 3.1 Pro Preview へ移行

### その他注目モデル

| モデル | Vision | 入力/MTok（目安） | 特徴 |
|---|---|---|---|
| **Meta Llama 4 Scout** | Yes | セルフホスト | 10Mコンテキスト、OSS最強クラス |
| **DeepSeek V3.2** | 限定的 | ~$0.14 | 685B MoE、超低コスト |
| **xAI Grok** | Yes | ~$0.20 | API最安クラス |

### 本プロジェクトでの推奨

| 用途 | 推奨モデル | 理由 |
|---|---|---|
| **スナップショット画像解析** | Gemini 3 Flash / Gemini 2.5 Flash | Vision精度トップクラスかつ低コスト（$0.30〜$0.50/MTok） |
| **コメント生成** | Claude Sonnet 4.6 | 配信コンテキストの理解と自然な日本語コメント生成に高品質が必要 |
| **コスト最小構成** | Gemini 2.5 Flash-Lite（画像） + GPT-5 Nano（コメント） | 全体で最も安価 |
| **品質最優先構成** | Claude Sonnet 4.6（画像+コメント一体） | 画像解析とコメント生成を1回のAPI呼び出しで処理 |

---

## 技術的な実現可能性と課題

> 本セクションはプロジェクト要件（社内向けYouTube Live、30分配信、1〜3分に1回のコメント投稿）に基づいて評価する。

### 実現可能（高確度）

| 項目 | 説明 |
|---|---|
| YouTube Live Chatへのコメント投稿 | YouTube Data API v3 `liveChatMessages.insert` で公式サポート。社内配信＋低頻度投稿のため技術的障壁はほぼない |
| ライブ配信の音声取得 | yt-dlp + ffmpeg でストリームURLを取得しリアルタイムでパイプ処理可能 |
| 音声文字起こし | 1〜3分に1回のコメント生成サイクルであれば、faster-whisper（チャンク処理、3〜5秒遅延）で十分。クラウドSTT（Deepgram等）ならさらに低遅延だが、本要件では不要 |
| 画面スナップショット取得 | ffmpeg で `-vf fps=1/N` により定期的なフレーム抽出が可能 |
| LLMによるコメント生成 | Claude Sonnet 4.6 / GPT-5.x 等のAPIで、コンテキストに基づくテキスト生成は成熟した技術 |
| マルチモーダル画像解析 | Claude Vision / GPT-5.x / Gemini 3.x で画像の内容理解が可能 |

### 技術的課題

#### 1. YouTube APIクォータ管理（影響度: 中）

- **課題**: デフォルトクォータ10,000 units/日。`insert` が50 units/回のため、1日の投稿可能数に制限がある
- **試算**（30分配信 × 1回/日の場合）:
  - コメント投稿: 20回 × 50 units = **1,000 units**
  - チャット取得（`streamList`）: ≈ **100 units**
  - 配信情報取得: ≈ **5 units**
  - 合計: **約1,100 units/日**（クォータの11%、余裕あり）
- **マルチBot時の注意**: Bot数 × 1,100 units。3Bot以上 or 1日複数配信ではクォータ逼迫の可能性
- **対策**: Bot毎に別のGCPプロジェクト/認証情報を使用。必要に応じてクォータ増枠申請

#### 2. LLM APIコストとレイテンシ（影響度: 中）

- **課題**: コメント生成のAPI呼び出しにかかるコストと応答時間が運用に影響する
- **コスト試算**（30分配信、90秒間隔でコメント生成、1回あたり入力2K/出力200トークン想定）:
  - Claude Sonnet 4.6: 20回 × (2K×$3 + 200×$15)/MTok ≈ **$0.07/配信**
  - Gemini 2.5 Flash（画像解析）: 60回 × 258トークン/画像 ≈ **$0.005/配信**
  - 月20配信でも合計 **$1.5/月程度**（コスト面の懸念は低い）
- **レイテンシ**: Claude Sonnet 4.6の応答は通常1〜3秒。コメント投稿間隔（90秒）に対して十分高速
- **対策**: API障害時のフォールバック（別プロバイダへの切り替え）を実装

#### 3. YouTube利用規約・スパム判定（影響度: 低〜中）

- **課題**: 自動コメント投稿はYouTubeのスパムフィルタに検出される可能性がある
- **本プロジェクトでのリスク評価**:
  - 社内配信（限定公開）+ 低頻度（1〜3分に1回）+ 配信者自身のチャンネル → **リスクは低い**
  - ただしYouTube API Services Terms of Serviceへの準拠は必須
- **対策**:
  - 同一内容の繰り返し投稿を避ける（LLMで毎回異なるコメントを生成）
  - 投稿間隔にランダムなジッターを加える（機械的パターンの回避）
  - YouTube APIの公式フローのみ使用（スクレイピング等は行わない）

#### 4. 配信の中断・再接続への対応（影響度: 中）

- **課題**: ネットワーク断、配信者の一時停止、yt-dlpのストリームURL期限切れ等により、配信の取得が中断する可能性がある
- **対策**:
  - ストリーム取得プロセスの死活監視と自動再接続（exponential backoff）
  - ffmpegプロセスの異常終了検知とリスタート
  - 再接続後のコンテキスト復旧（中断前の要約を保持）

#### 5. OAuth認証のトークン管理（影響度: 低）

- **課題**: YouTube APIのOAuthトークンは期限切れになるため、リフレッシュトークンの管理が必要
- **対策**: `google-auth-oauthlib` のトークンリフレッシュ機能で自動更新。初回認証のみ手動のブラウザフローが必要

#### 6. 長時間配信への拡張（影響度: 低 ※現要件では30分想定）

- **課題**: 1時間超の配信では文字起こし・画像解析データが蓄積し、LLMのコンテキストウィンドウに収まらない可能性
- **対策**:
  - スライディングウィンドウ方式（直近N分のみ保持）
  - 定期的な要約作成で古いデータを圧縮
- **備考**: 30分配信であれば文字起こし全量（約5,000〜8,000語）がコンテキストに収まるため、当面は単純な蓄積で問題ない

### リスクサマリー

| 課題 | 影響度 | 対応コスト | 優先度 |
|---|---|---|---|
| APIクォータ管理 | 中 | 低（設計時に考慮） | 高 |
| LLM APIコスト/レイテンシ | 中 | 低（月$1.5程度） | 中 |
| スパム判定 | 低〜中 | 低（投稿パターンの工夫） | 中 |
| 配信中断・再接続 | 中 | 中（再接続ロジック実装） | 高 |
| OAuth管理 | 低 | 低（ライブラリで対応） | 低 |
| 長時間配信対応 | 低 | 中（要約ロジック実装） | 低（将来対応） |

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
COMMENT_INTERVAL_SEC=90         # コメント投稿間隔（秒）※ 1〜3分に1回を想定
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
