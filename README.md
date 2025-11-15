# IEGUCHI - 不動産窓口営業アシスタント

> AIを活用した次世代不動産営業支援システム

## プロジェクト概要

**イエグチ（IEGUCHI）** は、不動産店舗での顧客対応を自動化するAIエージェントシステムです。音声対応・アバター表示・物件検索機能を統合し、高齢者にも分かりやすい接客体験を提供します。

### システム名
- **正式名称**: 不動産窓口営業アシスタント「イエグチ」
- **略称**: イエグチ / イエグチさん
- **英語表記**: IEGUCHI - Real Estate Front Desk Sales Assistant

### 主な特徴

- 🎙️ **音声対応**: OpenAI Whisper + ElevenLabsによる自然な会話
- 🤖 **AIアバター**: 表情豊かな静止画ベースのアバター表示
- 🏠 **物件検索**: GPT-4oによるインテリジェントな物件マッチング
- 🔒 **セキュア**: Supabase Auth + Row Level Securityによる多層セキュリティ
- 📊 **データ統合**: 自社DB + 外部サイト（アットホーム等）の物件情報統合

## クイックスタート

```bash
# リポジトリのクローン
git clone https://github.com/koma-ry1117/ieguchi.git
cd ieguchi

# 依存関係のインストール（pnpm推奨）
pnpm install

# 環境変数の設定
cp .env.example .env.local
# .env.local を編集してAPIキーを設定

# 開発サーバーの起動
pnpm dev
```

ブラウザで `http://localhost:3000` を開きます。

## 技術スタック

### フロントエンド
- **Next.js 15** (App Router) + TypeScript + React 19
- **Tailwind CSS** + shadcn/ui + Framer Motion
- **Zustand** (状態管理) + TanStack Query (データフェッチ)

### バックエンド・インフラ
- **Supabase** (PostgreSQL + Auth + Storage + Realtime)
- **Vercel** (ホスティング + Edge Functions)

### AI・音声処理
- **OpenAI GPT-4o** (会話・構造化・分析)
- **OpenAI Whisper** (音声認識)
- **ElevenLabs** (音声合成)
- **Vercel AI SDK** (AI統合・ストリーミング)

### セキュリティ・認証
- **Supabase Auth** (JWT + OAuth)
- **Row Level Security (RLS)** (データベースレベルアクセス制御)

詳細は [技術スタック詳細](./docs/technical-stack.md) を参照してください。

## プロジェクト構成

```
ieguchi/
├── README.md                       # このファイル
├── docs/                           # ドキュメント
│   ├── service-overview-diagram.md # サービス全体像（giyam様向け）⭐
│   ├── system-overview.md          # システム概要・キャラクター設定
│   ├── technical-stack.md          # 技術スタック詳細
│   ├── architecture.md             # アーキテクチャ設計
│   ├── database-design.md          # データベース設計
│   ├── api-design.md               # API設計
│   ├── cost-estimation.md          # コスト見積もり（御見積書準拠）
│   └── security.md                 # セキュリティ設計
├── src/                            # ソースコード（後で追加）
├── chrome-extension/               # Chrome拡張機能（後で追加）
└── package.json
```

## ドキュメント

### クライアント向け資料
- **[サービス全体像・アーキテクチャ図](./docs/service-overview-diagram.md)** - giyam様向け認識合わせ資料

### 技術ドキュメント
- [システム概要](./docs/system-overview.md) - キャラクター設定・接客フロー
- [技術スタック](./docs/technical-stack.md) - 使用技術の詳細説明
- [アーキテクチャ設計](./docs/architecture.md) - システム全体の構成
- [データベース設計](./docs/database-design.md) - テーブル設計・インデックス戦略
- [API設計](./docs/api-design.md) - エンドポイント仕様
- [コスト見積もり](./docs/cost-estimation.md) - 御見積書準拠のコスト試算
- [セキュリティ設計](./docs/security.md) - セキュリティ対策詳細

## ロードマップ

### Phase 1: 基盤構築（1-2ヶ月）
- [ ] Next.js + Supabase環境構築
- [ ] 認証システム実装
- [ ] 基本UI構築（shadcn/ui）

### Phase 2: AI機能実装（2-3ヶ月）
- [ ] OpenAI GPT-4o統合
- [ ] 音声認識・合成機能
- [ ] アバター表情切り替え

### Phase 3: 物件管理（1-2ヶ月）
- [ ] Chrome拡張機能開発
- [ ] 物件データベース構築
- [ ] 検索・フィルタリング機能

### Phase 4: 本番リリース（1ヶ月）
- [ ] テスト・デバッグ
- [ ] パフォーマンス最適化
- [ ] 本番デプロイ

## 開発環境

### 必須要件
- Node.js 20.x以上
- pnpm 9.x以上
- Git

### 推奨開発ツール
- **Cursor Pro** - AIペアプログラミングIDE
- **Claude Pro (Max Plan)** - 技術相談・コードレビュー
- **Playwright** - E2Eテスト

## ライセンス

このプロジェクトはプライベートプロジェクトです。

## 開発者

- **依頼者**: giyamさん（医療理科学ガラス問屋・不動産事業）
- **開発者**: koma-ry1117

## お問い合わせ

プロジェクトに関する質問や提案は、GitHubのIssuesでお願いします。

---

**開発開始日**: 2025年11月15日
