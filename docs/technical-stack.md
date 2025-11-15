# 技術スタック詳細

## 全体構成

このプロジェクトは**モダンなフルスタック構成**で、以下の技術を採用しています。

```
フロントエンド (Next.js + React)
         ↕
   認証・認可層 (Supabase Auth)
         ↕
AI・音声処理層 (OpenAI + ElevenLabs)
         ↕
  データベース層 (Supabase PostgreSQL)
         ↕
Chrome拡張機能 (物件取り込み)
```

---

## 1. フロントエンド

### コア技術

#### Next.js v15 (App Router)
- **選定理由**:
  - React Server Componentsによる高速化
  - ファイルベースルーティング
  - サーバーレスアーキテクチャとの親和性
  - Vercelとの完璧な統合
- **使用機能**:
  - App Router（`app/` ディレクトリ）
  - Server Actions（フォーム送信・データ更新）
  - Route Handlers（APIエンドポイント）
  - Metadata API（SEO対策）
  - Image Optimization

#### TypeScript
- **選定理由**:
  - 型安全性による開発効率向上
  - AIコード生成との相性が良い
  - 大規模化時のメンテナンス性
  - チーム開発時のコミュニケーションコスト削減

#### React 19
- **使用機能**:
  - Hooks（useState, useEffect, useCallback等）
  - Context API（グローバル状態の軽量管理）
  - Suspense（非同期データローディング）

#### Tailwind CSS
- **選定理由**:
  - ユーティリティファーストで開発速度向上
  - デザインの一貫性確保
  - バンドルサイズ最適化（PurgeCSS）
  - shadcn/uiとの完璧な統合

### UI/UXライブラリ

#### shadcn/ui
- **選定理由**:
  - 高品質なアクセシブルコンポーネント
  - Radix UIベースで堅牢
  - カスタマイズ性が高い
  - コピー&ペーストで使える
- **使用コンポーネント**:
  - Button, Input, Select, Dialog
  - Dropdown Menu, Popover, Tooltip
  - Card, Avatar, Badge
  - Form, Table, Tabs

#### Framer Motion
- **用途**: アバター表情切り替えアニメーション
- **使用機能**:
  - `motion.img` による画像アニメーション
  - `AnimatePresence` による入退場アニメーション
  - スプリングアニメーション（自然な動き）

#### React Hook Form
- **用途**: フォーム管理
- **使用機能**:
  - バリデーション統合（Zod）
  - パフォーマンス最適化
  - エラーハンドリング

#### Zod
- **用途**: スキーマバリデーション
- **使用例**:
  - フォーム入力検証
  - APIレスポンス検証
  - 環境変数検証

---

## 2. バックエンド・インフラ

### データベース・ストレージ

#### Supabase
**選定理由**:
- PostgreSQL + Auth + Storage + Realtime が統合
- オープンソースで将来的な移行可能性
- RLS（Row Level Security）による強力なセキュリティ
- REST API + Realtime subscriptions

**使用機能**:

1. **PostgreSQL データベース**
   - リレーショナルデータベース
   - JSON型サポート
   - 全文検索（FTS）
   - トリガー・関数

2. **Row Level Security (RLS)**
   - データベースレベルでのアクセス制御
   - ユーザーロール別の権限管理
   - SQLポリシーによる柔軟な設定

3. **Realtime subscriptions**
   - データベース変更のリアルタイム通知
   - WebSocketベースの双方向通信
   - 複数クライアント間の同期

4. **Storage**
   - 物件画像ストレージ
   - 音声キャッシュストレージ
   - アバター画像ホスティング
   - HTMLスナップショット保存

### ホスティング・デプロイ

#### Vercel
**選定理由**:
- Next.jsとの完璧な統合
- 自動スケーリング
- グローバルCDN
- プレビューデプロイ（PRごと）
- ゼロコンフィグデプロイ

**使用機能**:

1. **Vercel Edge Functions**
   - エッジコンピューティング
   - 低レイテンシ
   - グローバル展開

2. **Vercel Edge Middleware**
   - レート制限
   - リダイレクト・リライト
   - 認証チェック
   - セキュリティヘッダー

3. **Vercel Analytics**
   - Webパフォーマンス分析
   - Core Web Vitals測定
   - リアルタイムアクセス解析

---

## 3. 認証・認可システム

### Supabase Auth

**認証方式**:
- メール/パスワード認証
- OAuth認証（Google, GitHub等）サポート
- マジックリンク認証（パスワードレス）

**セッション管理**:
- JWT (JSON Web Token) ベースのステートレス認証
- Refresh Token による自動セッション更新
- マルチセッション管理

**セキュリティ機能**:
- パスワードハッシュ化（bcrypt）
- パスワードリセット機能
- メール確認機能
- セッション有効期限管理

### 権限管理（Row Level Security）

```sql
-- オペレーター：自分が取り込んだ物件の編集権限
CREATE POLICY "operator_own_properties"
ON properties FOR UPDATE
USING (auth.uid() = created_by);

-- 管理者：全物件の編集・削除権限
CREATE POLICY "admin_all_properties"
ON properties FOR ALL
USING (
  (SELECT role FROM auth.users WHERE id = auth.uid()) = 'admin'
);

-- 顧客：公開物件の閲覧のみ
CREATE POLICY "customer_view_public"
ON properties FOR SELECT
USING (is_public = true);
```

### ユーザーロール

| ロール | 権限 |
|--------|------|
| customer | 物件閲覧、会話履歴閲覧（自分のみ） |
| operator | 物件取り込み、自分の物件編集、顧客対応履歴確認 |
| admin | 全物件編集・削除、ユーザー管理、システム設定 |

---

## 4. AI・LLM統合（OpenAI統一）

### 会話AI

#### OpenAI GPT-4o
**用途**:
- メイン会話エンジン
- 顧客対応・会話処理
- 物件推薦・マッチング
- 感情分析（アバター表情選択用）
- HTML構造化データ抽出
- 物件説明文生成

**モデル仕様**:
- コンテキスト長: 128K トークン
- 入力価格: $5/1M トークン
- 出力価格: $15/1M トークン
- マルチモーダル対応（テキスト・画像）

**使用例**:
```typescript
import { openai } from '@ai-sdk/openai';
import { generateText, streamText } from 'ai';

// 会話生成
const { text } = await generateText({
  model: openai('gpt-4o'),
  system: 'あなたは不動産営業のイエグチさんです...',
  prompt: userMessage,
});

// ストリーミング会話
const { textStream } = streamText({
  model: openai('gpt-4o'),
  messages: conversationHistory,
});
```

### AIフレームワーク

#### Vercel AI SDK
**選定理由**:
- Next.jsとの完璧な統合
- OpenAI統合が簡単
- ストリーミングレスポンス対応
- Function Calling対応
- エッジランタイム対応

**使用機能**:

1. **ストリーミングレスポンス**
   ```typescript
   import { StreamingTextResponse } from 'ai';

   return new StreamingTextResponse(stream);
   ```

2. **Function Calling（物件検索実行）**
   ```typescript
   const tools = {
     searchProperties: {
       description: '物件を検索する',
       parameters: z.object({
         area: z.string(),
         rent: z.object({
           min: z.number(),
           max: z.number(),
         }),
         layout: z.string(),
       }),
       execute: async (params) => {
         return await searchProperties(params);
       },
     },
   };
   ```

3. **Tool Calling（複数ツール統合）**
   - 物件検索
   - 画像取得
   - 周辺情報取得
   - 類似物件検索

### ベクトル検索・RAG

#### Supabase Vector (pgvector)
**用途**: セマンティック物件検索
- ベクトルデータベース
- 類似物件検索
- 顧客の希望に近い物件の発見

#### OpenAI Embeddings (text-embedding-3-small)
**用途**: テキストのベクトル化
- 物件説明文のベクトル化
- 顧客の希望条件のベクトル化
- コサイン類似度による検索

**価格**: $0.02/1M トークン

**実装例**:
```typescript
import { embeddings } from '@ai-sdk/openai';

// テキストをベクトル化
const vector = await embeddings.embed({
  model: 'text-embedding-3-small',
  input: propertyDescription,
});

// Supabaseで類似検索
const { data } = await supabase.rpc('match_properties', {
  query_embedding: vector,
  match_threshold: 0.8,
  match_count: 10,
});
```

---

## 5. 音声処理

### 音声認識 (Speech-to-Text)

#### OpenAI Whisper API
**仕様**:
- 価格: $0.006/分
- 対応形式: mp3, mp4, wav, webm等
- 高精度な日本語認識
- タイムスタンプ付き文字起こし

**使用例**:
```typescript
import { openai } from 'openai';

const transcription = await openai.audio.transcriptions.create({
  file: audioFile,
  model: 'whisper-1',
  language: 'ja',
});
```

#### Web Audio API
**用途**: ブラウザでのマイク入力処理
- マイク入力の取得
- 音声レベルメーター
- リアルタイム波形表示

#### MediaRecorder API
**用途**: 音声録音・エンコーディング
- ブラウザで音声録音
- WebM形式で保存
- Blob生成

### 音声合成 (Text-to-Speech)

#### ElevenLabs API
**選定理由**:
- 高品質な日本語音声合成
- 感情表現が豊か
- カスタム音声作成可能
- ストリーミング再生対応

**価格**:
- Starter: 30,000文字/月 - $5
- Creator: 100,000文字/月 - $11
- Pro: 500,000文字/月 - $99

**使用例**:
```typescript
import { ElevenLabsClient } from 'elevenlabs';

const audio = await elevenlabs.textToSpeech({
  voice_id: 'voice_id_here',
  text: response,
  model_id: 'eleven_multilingual_v2',
});
```

#### 音声キャッシング戦略
よくある応答を事前生成してSupabase Storageに保存

```typescript
// キャッシュ確認
const cachedAudio = await supabase.storage
  .from('audio-cache')
  .download(`${hash(text)}.mp3`);

if (cachedAudio) {
  return cachedAudio;
}

// キャッシュがなければ生成
const audio = await generateSpeech(text);
await supabase.storage
  .from('audio-cache')
  .upload(`${hash(text)}.mp3`, audio);
```

---

## 6. メール送信機能 ⭐ NEW

### Resend

**選定理由**:
- Next.jsとの完璧な統合
- シンプルなAPI
- 無料枠が充実（月100通まで無料）
- 配信率が高い
- Vercelとの相性が良い

**価格**:
- Free: 100通/月 - 無料
- Pro: 50,000通/月 - $20
- Business: カスタム

**使用例**:
```typescript
import { Resend } from 'resend';

const resend = new Resend(process.env.RESEND_API_KEY);

await resend.emails.send({
  from: 'イエグチ <noreply@ieguchi.jp>',
  to: customer.email,
  subject: '内見予約が確定しました',
  react: ViewingConfirmationEmail({
    customerName: customer.name,
    propertyTitle: property.title,
    viewingDate: '2025/11/20 14:00',
  }),
});
```

### React Email

**用途**: 美しいHTMLメールテンプレート作成

**選定理由**:
- Reactコンポーネントでメール作成
- レスポンシブデザイン対応
- プレビュー機能
- TypeScript完全サポート

**テンプレート例**:
```typescript
// emails/viewing-confirmation.tsx
import {
  Body, Container, Head, Heading, Html,
  Preview, Section, Text, Button
} from '@react-email/components';

export default function ViewingConfirmationEmail({
  customerName,
  propertyTitle,
  viewingDate
}) {
  return (
    <Html>
      <Head />
      <Preview>内見予約が確定しました</Preview>
      <Body style={main}>
        <Container style={container}>
          <Heading style={h1}>
            内見予約確定のお知らせ
          </Heading>
          <Text style={text}>
            {customerName} 様
          </Text>
          <Text style={text}>
            以下の物件の内見予約が確定しました。
          </Text>
          <Section style={propertyInfo}>
            <Text><strong>物件名:</strong> {propertyTitle}</Text>
            <Text><strong>内見日時:</strong> {viewingDate}</Text>
          </Section>
          <Button
            href="https://ieguchi.jp/viewings"
            style={button}
          >
            予約詳細を確認
          </Button>
        </Container>
      </Body>
    </Html>
  );
}
```

### メール種別

#### 1. 内見予約確定メール
- 顧客への確認メール
- オペレーターへの通知
- カレンダー招待（.ics）添付

#### 2. システムお知らせメール
- 新着物件のお知らせ
- メンテナンス通知
- 重要な変更の連絡

#### 3. 問い合わせ確認メール
- 問い合わせ受付の自動返信
- 担当者アサイン通知

#### 4. 認証関連メール
- パスワードリセット（Supabase Auth）
- メール確認
- アカウント登録完了

### カレンダー招待（.ics）生成

```typescript
import ical from 'ical-generator';

const calendar = ical({
  name: '内見予約',
  timezone: 'Asia/Tokyo'
});

calendar.createEvent({
  start: new Date('2025-11-20T14:00:00+09:00'),
  end: new Date('2025-11-20T15:00:00+09:00'),
  summary: `物件内見: ${property.title}`,
  description: `住所: ${property.address}`,
  location: property.address,
  url: `https://ieguchi.jp/properties/${property.id}`,
  organizer: {
    name: 'イエグチ',
    email: 'noreply@ieguchi.jp',
  },
  attendees: [
    {
      name: customer.name,
      email: customer.email,
    },
  ],
});

// .icsファイルをメールに添付
await resend.emails.send({
  from: 'イエグチ <noreply@ieguchi.jp>',
  to: customer.email,
  subject: '内見予約が確定しました',
  react: ViewingConfirmationEmail({ ... }),
  attachments: [
    {
      filename: 'viewing.ics',
      content: calendar.toString(),
    },
  ],
});
```

---

## 7. アバター表示（静止画ベース）

### アバター画像管理

**静止画セット（5種類）**:
```
public/avatars/
├── neutral.png     # 通常
├── happy.png       # 喜び
├── thinking.png    # 考え中
├── apologetic.png  # 謝罪
└── excited.png     # 興奮
```

**画像仕様**:
- 形式: PNG（透過背景）
- サイズ: 512x512px 推奨
- 最適化: TinyPNGで圧縮

### 表情切り替えシステム

#### GPT-4o感情分析
```typescript
const analyzeEmotion = async (response: string) => {
  const { object } = await generateObject({
    model: openai('gpt-4o'),
    schema: z.object({
      emotion: z.enum(['neutral', 'happy', 'thinking', 'apologetic', 'excited']),
      confidence: z.number().min(0).max(1),
      reason: z.string(),
    }),
    prompt: `以下のレスポンスから感情を判定してください：${response}`,
  });

  return object;
};
```

#### React状態管理（Zustand）
```typescript
import { create } from 'zustand';

interface AvatarStore {
  emotion: EmotionType;
  setEmotion: (emotion: EmotionType) => void;
}

export const useAvatarStore = create<AvatarStore>((set) => ({
  emotion: 'neutral',
  setEmotion: (emotion) => set({ emotion }),
}));
```

#### Framer Motion アニメーション
```typescript
import { motion, AnimatePresence } from 'framer-motion';

<AnimatePresence mode="wait">
  <motion.img
    key={emotion}
    src={`/avatars/${emotion}.png`}
    initial={{ opacity: 0, scale: 0.95 }}
    animate={{ opacity: 1, scale: 1 }}
    exit={{ opacity: 0, scale: 0.95 }}
    transition={{ duration: 0.3, ease: 'easeInOut' }}
  />
</AnimatePresence>
```

---

## 7. Chrome拡張機能（最小構成）

### 目的
- 表示中のページのHTMLを取得→アプリに送信のみ
- 人手によるワンクリック取り込み
- 法的リスク軽減のための証跡記録

### 技術スタック

#### Manifest V3
最新のChrome拡張仕様

#### TypeScript
型安全な拡張機能開発

#### Vanilla JavaScript
フレームワーク不使用で軽量化

### 構成要素

```
chrome-extension/
├── manifest.json              # 拡張機能設定
├── src/
│   ├── background.ts          # Service Worker
│   ├── content.ts             # Content Script
│   ├── popup/
│   │   ├── popup.html         # ポップアップUI
│   │   ├── popup.ts           # ポップアップロジック
│   │   └── popup.css          # スタイル
│   └── types.ts               # 型定義
└── dist/                      # ビルド出力
```

#### manifest.json
```json
{
  "manifest_version": 3,
  "name": "IEGUCHI Property Importer",
  "version": "1.0.0",
  "permissions": [
    "activeTab",
    "storage"
  ],
  "host_permissions": [
    "https://www.athome.co.jp/*"
  ],
  "background": {
    "service_worker": "background.js"
  },
  "content_scripts": [
    {
      "matches": ["https://www.athome.co.jp/*"],
      "js": ["content.js"]
    }
  ],
  "action": {
    "default_popup": "popup.html"
  }
}
```

#### Content Script（HTML取得）
```typescript
// content.ts
chrome.runtime.onMessage.addListener((request, sender, sendResponse) => {
  if (request.action === 'getHTML') {
    const html = document.documentElement.outerHTML;
    const url = window.location.href;

    sendResponse({ html, url });
  }
});
```

#### Background Service Worker（API送信）
```typescript
// background.ts
chrome.action.onClicked.addListener(async (tab) => {
  const [{ result }] = await chrome.scripting.executeScript({
    target: { tabId: tab.id },
    func: () => document.documentElement.outerHTML,
  });

  // Next.js APIに送信
  const response = await fetch('https://ieguchi.vercel.app/api/properties/import', {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      'Authorization': `Bearer ${token}`,
    },
    body: JSON.stringify({
      html: result,
      url: tab.url,
    }),
  });
});
```

---

## 8. 状態管理・データフェッチ

### グローバル状態管理

#### Zustand
**選定理由**:
- Reduxより軽量・シンプル
- TypeScript完全対応
- Reactフックベース
- DevTools対応

**管理する状態**:
- 会話履歴
- 現在の物件フィルタ
- アバター感情状態
- 認証状態（ユーザー情報・セッション）
- UI状態（モーダル・サイドバー等）

**ストア例**:
```typescript
import { create } from 'zustand';
import { persist } from 'zustand/middleware';

interface ChatStore {
  messages: Message[];
  addMessage: (message: Message) => void;
  clearMessages: () => void;
}

export const useChatStore = create<ChatStore>()(
  persist(
    (set) => ({
      messages: [],
      addMessage: (message) =>
        set((state) => ({ messages: [...state.messages, message] })),
      clearMessages: () => set({ messages: [] }),
    }),
    {
      name: 'chat-storage',
    }
  )
);
```

### サーバーステート管理

#### TanStack Query (React Query)
**選定理由**:
- サーバーデータのキャッシング
- 自動再取得・バックグラウンド更新
- 楽観的更新
- 無限スクロール対応
- エラーハンドリング・リトライ

**使用例**:
```typescript
import { useQuery, useMutation, useQueryClient } from '@tanstack/react-query';

// データフェッチ
const { data, isLoading } = useQuery({
  queryKey: ['properties', filters],
  queryFn: () => fetchProperties(filters),
  staleTime: 5 * 60 * 1000, // 5分間キャッシュ
});

// データ更新
const mutation = useMutation({
  mutationFn: updateProperty,
  onSuccess: () => {
    queryClient.invalidateQueries(['properties']);
  },
});
```

---

## 9. テスト・品質保証

### ユニットテスト

#### Vitest
**選定理由**:
- Jest互換API
- Viteベースで高速
- ESM完全対応
- TypeScript完全サポート

#### Testing Library (React Testing Library)
**用途**: Reactコンポーネントテスト

#### MSW (Mock Service Worker)
**用途**: API モック

**テスト例**:
```typescript
import { describe, it, expect } from 'vitest';
import { render, screen } from '@testing-library/react';

describe('PropertyCard', () => {
  it('物件情報が正しく表示される', () => {
    render(<PropertyCard property={mockProperty} />);
    expect(screen.getByText('2LDK')).toBeInTheDocument();
  });
});
```

### E2Eテスト

#### Playwright
**選定理由**:
- クロスブラウザテスト
- 高速・安定
- 音声入力シミュレーション可能
- Chrome拡張機能テスト対応

**テスト例**:
```typescript
import { test, expect } from '@playwright/test';

test('物件検索フロー', async ({ page }) => {
  await page.goto('http://localhost:3000');
  await page.click('[aria-label="音声入力開始"]');
  await page.fill('input[name="条件"]', '文京区 2LDK');
  await page.click('button:has-text("検索")');

  await expect(page.locator('.property-card')).toHaveCount(10);
});
```

### 型チェック・リンター

- **TypeScript**: 静的型チェック
- **ESLint**: コード品質チェック
- **Prettier**: コードフォーマット
- **Husky**: Git pre-commit hooks

---

## 10. モニタリング・分析

### エラートラッキング

#### Sentry
**機能**:
- フロントエンドエラー監視
- APIエラー監視
- パフォーマンスモニタリング
- ユーザーセッション再生
- ソースマップ対応

**価格**:
- Developer: 無料（5,000イベント/月）
- Team: $26/月（50,000イベント/月）

### アナリティクス

#### Vercel Analytics
- Core Web Vitals測定
- リアルタイムアクセス解析
- デバイス・ブラウザ分析

#### PostHog（オプション）
- イベントトラッキング
- ファネル分析
- セッションリプレイ
- A/Bテスト

### ログ管理

#### Axiom
- リアルタイムログ分析
- Vercel統合
- ダッシュボード作成

#### Supabase Logs
- データベースクエリログ
- 遅いクエリの検出

---

## 11. 開発ツール・ワークフロー

### AI駆動開発

#### Cursor
- AIペアプログラミングIDE
- コード補完・生成
- リファクタリング提案
- 価格: $20/月

#### Claude Pro (Max Plan)
- 技術相談・アーキテクチャ設計
- コードレビュー
- ドキュメント作成
- 複雑な問題解決
- 価格: $100/月（¥15,000）

**Max Planの価値**:
- 通常のClaude Proより5倍多いメッセージ送信
- より長い会話コンテキスト
- 複雑な技術問題の解決に必須

### バージョン管理・CI/CD

#### GitHub Actions
```yaml
name: CI/CD

on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: pnpm/action-setup@v2
      - run: pnpm install
      - run: pnpm type-check
      - run: pnpm lint
      - run: pnpm test

  deploy:
    needs: test
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    steps:
      - uses: actions/checkout@v4
      - uses: amondnet/vercel-action@v25
```

### 開発環境

#### pnpm
**選定理由**:
- npm/yarnより高速
- ディスク容量節約
- モノレポ対応

---

## 12. セキュリティ

### 認証・認可
- Supabase Auth（JWT）
- Row Level Security (RLS)
- Refresh Token

### API保護
- CORS設定
- Rate Limiting
- API Key管理
- Request Validation (Zod)

### セキュリティヘッダー
```typescript
// middleware.ts
export function middleware(request: NextRequest) {
  const response = NextResponse.next();

  response.headers.set('X-Frame-Options', 'DENY');
  response.headers.set('X-Content-Type-Options', 'nosniff');
  response.headers.set('Referrer-Policy', 'strict-origin-when-cross-origin');
  response.headers.set(
    'Content-Security-Policy',
    "default-src 'self'; script-src 'self' 'unsafe-eval' 'unsafe-inline';"
  );

  return response;
}
```

### データ保護
- 暗号化（機密データ）
- SQLインジェクション対策
- XSS対策
- CSRF対策

---

## 技術スタック選定理由まとめ

### なぜこのスタックなのか

1. **Next.js + Vercel**
   - サーバーレスアーキテクチャで運用コスト最小化
   - 自動スケーリング対応
   - エッジコンピューティングで低レイテンシ

2. **Supabase**
   - PostgreSQL + Auth + Storage + Realtime が統合
   - オープンソースで将来的な移行可能性
   - RLSによるセキュアなデータアクセス

3. **OpenAI統一**
   - GPT-4o単体で会話・構造化・分析すべてカバー
   - Whisper APIで高精度な日本語音声認識
   - コスト管理が容易（単一プロバイダー）

4. **Vercel AI SDK**
   - Next.jsとの完璧な統合
   - ストリーミング対応
   - Function Calling対応

5. **TypeScript**
   - 型安全性による開発効率向上
   - AIコード生成との相性が良い
   - 大規模化時のメンテナンス性

6. **Claude Pro (Max Plan)**
   - 学生開発者の技術メンター
   - 複雑な設計判断のサポート
   - コードレビュー・最適化提案

---

次のドキュメント: [アーキテクチャ設計](./architecture.md)
