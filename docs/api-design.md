# API設計

## API構成

Next.js App Router の **Route Handlers** を使用

```
app/api/
├── auth/
│   ├── login/route.ts
│   ├── logout/route.ts
│   └── refresh/route.ts
├── chat/
│   └── route.ts
├── voice/
│   ├── transcribe/route.ts
│   └── synthesize/route.ts
├── properties/
│   ├── route.ts
│   ├── [id]/route.ts
│   ├── import/route.ts
│   └── search/route.ts
├── customers/
│   └── preferences/route.ts
└── admin/
    ├── users/route.ts
    └── logs/route.ts
```

---

## 共通仕様

### ベースURL
```
開発環境: http://localhost:3000/api
本番環境: https://ieguchi.vercel.app/api
```

### 認証方式
- **Bearer Token** (JWT)
- Supabase Authで発行

```http
Authorization: Bearer <access_token>
```

### レスポンス形式

#### 成功レスポンス
```json
{
  "success": true,
  "data": { ... }
}
```

#### エラーレスポンス
```json
{
  "success": false,
  "error": {
    "code": "ERROR_CODE",
    "message": "エラーメッセージ",
    "details": { ... }
  }
}
```

### HTTPステータスコード
| コード | 説明 |
|--------|------|
| 200 | 成功 |
| 201 | 作成成功 |
| 400 | リクエストエラー |
| 401 | 認証エラー |
| 403 | 権限エラー |
| 404 | リソースが見つからない |
| 429 | レート制限超過 |
| 500 | サーバーエラー |

---

## 認証API

### POST /api/auth/login

**説明**: メール/パスワードでログイン

**リクエスト**:
```json
{
  "email": "user@example.com",
  "password": "password123"
}
```

**レスポンス**:
```json
{
  "success": true,
  "data": {
    "user": {
      "id": "uuid",
      "email": "user@example.com",
      "role": "customer"
    },
    "session": {
      "access_token": "...",
      "refresh_token": "...",
      "expires_at": 1234567890
    }
  }
}
```

**実装例**:
```typescript
// app/api/auth/login/route.ts
import { createRouteHandlerClient } from '@supabase/auth-helpers-nextjs';
import { cookies } from 'next/headers';
import { NextResponse } from 'next/server';

export async function POST(request: Request) {
  const { email, password } = await request.json();

  const supabase = createRouteHandlerClient({ cookies });

  const { data, error } = await supabase.auth.signInWithPassword({
    email,
    password,
  });

  if (error) {
    return NextResponse.json(
      { success: false, error: { code: 'AUTH_FAILED', message: error.message } },
      { status: 401 }
    );
  }

  return NextResponse.json({
    success: true,
    data: {
      user: data.user,
      session: data.session,
    },
  });
}
```

---

### POST /api/auth/logout

**説明**: ログアウト

**レスポンス**:
```json
{
  "success": true,
  "data": {
    "message": "ログアウトしました"
  }
}
```

---

### POST /api/auth/refresh

**説明**: アクセストークンのリフレッシュ

**リクエスト**:
```json
{
  "refresh_token": "..."
}
```

**レスポンス**:
```json
{
  "success": true,
  "data": {
    "access_token": "...",
    "expires_at": 1234567890
  }
}
```

---

## AI会話API

### POST /api/chat

**説明**: AI会話エンドポイント（ストリーミング対応）

**認証**: 必須

**リクエスト**:
```json
{
  "messages": [
    {
      "role": "user",
      "content": "文京区で2LDKの物件を探しています"
    }
  ],
  "conversation_id": "uuid" // 既存の会話の場合
}
```

**レスポンス**: Server-Sent Events (SSE)

```
data: {"type":"text","content":"かしこまりました。"}
data: {"type":"text","content":"文京区で2LDK"}
data: {"type":"text","content":"の物件ですね。"}
data: {"type":"emotion","emotion":"thinking"}
data: {"type":"function_call","name":"searchProperties","args":{"area":"文京区","layout":"2LDK"}}
data: {"type":"function_result","result":[...]}
data: {"type":"text","content":"以下の物件がございます..."}
data: {"type":"emotion","emotion":"happy"}
data: {"type":"done"}
```

**実装例**:
```typescript
// app/api/chat/route.ts
import { openai } from '@ai-sdk/openai';
import { streamText } from 'ai';
import { createRouteHandlerClient } from '@supabase/auth-helpers-nextjs';
import { cookies } from 'next/headers';

export async function POST(request: Request) {
  const { messages, conversation_id } = await request.json();

  const supabase = createRouteHandlerClient({ cookies });

  // 認証チェック
  const { data: { user }, error } = await supabase.auth.getUser();
  if (error || !user) {
    return new Response('Unauthorized', { status: 401 });
  }

  // 会話履歴の保存・取得
  let conversationId = conversation_id;
  if (!conversationId) {
    const { data } = await supabase
      .from('conversations')
      .insert({
        customer_id: user.id,
      })
      .select()
      .single();
    conversationId = data.id;
  }

  // AI生成
  const result = await streamText({
    model: openai('gpt-4o'),
    system: `あなたは不動産営業のイエグチさんです。
丁寧で親しみやすく、高齢者にも分かりやすい説明を心がけてください。`,
    messages,
    tools: {
      searchProperties: {
        description: '物件を検索する',
        parameters: z.object({
          area: z.string().optional(),
          rent_min: z.number().optional(),
          rent_max: z.number().optional(),
          layout: z.string().optional(),
        }),
        execute: async (params) => {
          const { data } = await supabase
            .from('properties')
            .select('*')
            .eq('area', params.area)
            .gte('rent', params.rent_min || 0)
            .lte('rent', params.rent_max || 999999999)
            .eq('layout', params.layout)
            .eq('is_public', true)
            .limit(10);

          return data;
        },
      },
    },
  });

  // メッセージをDBに保存（非同期）
  result.onFinish(async ({ text }) => {
    await supabase.from('messages').insert([
      {
        conversation_id: conversationId,
        role: 'user',
        content: messages[messages.length - 1].content,
      },
      {
        conversation_id: conversationId,
        role: 'assistant',
        content: text,
      },
    ]);
  });

  return result.toAIStreamResponse();
}
```

---

## 音声処理API

### POST /api/voice/transcribe

**説明**: 音声→テキスト変換（OpenAI Whisper）

**認証**: 必須

**リクエスト**: multipart/form-data
```
audio: <audio file (mp3, wav, webm)>
```

**レスポンス**:
```json
{
  "success": true,
  "data": {
    "text": "文京区で2LDKの物件を探しています",
    "duration": 3.5
  }
}
```

**実装例**:
```typescript
// app/api/voice/transcribe/route.ts
import { NextResponse } from 'next/server';
import OpenAI from 'openai';

const openai = new OpenAI({
  apiKey: process.env.OPENAI_API_KEY,
});

export async function POST(request: Request) {
  const formData = await request.formData();
  const audio = formData.get('audio') as File;

  if (!audio) {
    return NextResponse.json(
      { success: false, error: { code: 'MISSING_AUDIO', message: '音声ファイルが必要です' } },
      { status: 400 }
    );
  }

  const transcription = await openai.audio.transcriptions.create({
    file: audio,
    model: 'whisper-1',
    language: 'ja',
  });

  return NextResponse.json({
    success: true,
    data: {
      text: transcription.text,
    },
  });
}
```

---

### POST /api/voice/synthesize

**説明**: テキスト→音声変換（ElevenLabs）+ キャッシング

**認証**: 必須

**リクエスト**:
```json
{
  "text": "かしこまりました。文京区で2LDKの物件ですね。"
}
```

**レスポンス**: audio/mpeg (音声バイナリ)

**実装例**:
```typescript
// app/api/voice/synthesize/route.ts
import { NextResponse } from 'next/server';
import { ElevenLabsClient } from 'elevenlabs';
import { createRouteHandlerClient } from '@supabase/auth-helpers-nextjs';
import { cookies } from 'next/headers';
import crypto from 'crypto';

const elevenlabs = new ElevenLabsClient({
  apiKey: process.env.ELEVENLABS_API_KEY,
});

export async function POST(request: Request) {
  const { text } = await request.json();

  const supabase = createRouteHandlerClient({ cookies });

  // テキストのハッシュ値を生成
  const hash = crypto.createHash('sha256').update(text).digest('hex');
  const cacheKey = `audio-cache/${hash}.mp3`;

  // キャッシュ確認
  const { data: cachedAudio } = await supabase.storage
    .from('audio-cache')
    .download(cacheKey);

  if (cachedAudio) {
    return new Response(cachedAudio, {
      headers: {
        'Content-Type': 'audio/mpeg',
      },
    });
  }

  // 音声生成
  const audio = await elevenlabs.textToSpeech.convert('voice_id_here', {
    text,
    model_id: 'eleven_multilingual_v2',
  });

  const audioBuffer = Buffer.from(await audio.arrayBuffer());

  // キャッシュ保存
  await supabase.storage.from('audio-cache').upload(cacheKey, audioBuffer, {
    contentType: 'audio/mpeg',
  });

  return new Response(audioBuffer, {
    headers: {
      'Content-Type': 'audio/mpeg',
    },
  });
}
```

---

## 物件API

### GET /api/properties

**説明**: 物件一覧取得

**認証**: 必須

**クエリパラメータ**:
| パラメータ | 型 | 説明 |
|-----------|-----|------|
| area | string | エリア（文京区等） |
| rent_min | number | 最低賃料 |
| rent_max | number | 最高賃料 |
| layout | string | 間取り（1K, 2LDK等） |
| building_type | string | 建物種別 |
| page | number | ページ番号（デフォルト: 1） |
| limit | number | 取得件数（デフォルト: 20） |

**レスポンス**:
```json
{
  "success": true,
  "data": {
    "properties": [
      {
        "id": "uuid",
        "title": "文京区本郷 2LDK マンション",
        "address": "東京都文京区本郷1-2-3",
        "area": "文京区",
        "rent": 150000,
        "layout": "2LDK",
        "building_type": "マンション",
        "floor_area": 55.5,
        "station": "本郷三丁目駅",
        "distance_from_station": 5,
        "images": [
          {
            "url": "https://..."
          }
        ]
      }
    ],
    "pagination": {
      "total": 100,
      "page": 1,
      "limit": 20,
      "total_pages": 5
    }
  }
}
```

**実装例**:
```typescript
// app/api/properties/route.ts
import { createRouteHandlerClient } from '@supabase/auth-helpers-nextjs';
import { cookies } from 'next/headers';
import { NextResponse } from 'next/server';

export async function GET(request: Request) {
  const { searchParams } = new URL(request.url);

  const area = searchParams.get('area');
  const rent_min = Number(searchParams.get('rent_min')) || 0;
  const rent_max = Number(searchParams.get('rent_max')) || 999999999;
  const layout = searchParams.get('layout');
  const page = Number(searchParams.get('page')) || 1;
  const limit = Number(searchParams.get('limit')) || 20;

  const supabase = createRouteHandlerClient({ cookies });

  let query = supabase
    .from('properties')
    .select('*, property_images(*)', { count: 'exact' })
    .eq('is_public', true)
    .gte('rent', rent_min)
    .lte('rent', rent_max)
    .range((page - 1) * limit, page * limit - 1);

  if (area) query = query.eq('area', area);
  if (layout) query = query.eq('layout', layout);

  const { data, error, count } = await query;

  if (error) {
    return NextResponse.json(
      { success: false, error: { code: 'DB_ERROR', message: error.message } },
      { status: 500 }
    );
  }

  return NextResponse.json({
    success: true,
    data: {
      properties: data,
      pagination: {
        total: count,
        page,
        limit,
        total_pages: Math.ceil(count / limit),
      },
    },
  });
}
```

---

### GET /api/properties/[id]

**説明**: 物件詳細取得

**認証**: 必須

**レスポンス**:
```json
{
  "success": true,
  "data": {
    "id": "uuid",
    "title": "...",
    "address": "...",
    "rent": 150000,
    "layout": "2LDK",
    "description": "...",
    "images": [...]
  }
}
```

---

### POST /api/properties/import

**説明**: Chrome拡張からの物件取り込み

**認証**: 必須（operator or admin）

**リクエスト**:
```json
{
  "html": "<html>...</html>",
  "url": "https://www.athome.co.jp/..."
}
```

**レスポンス**:
```json
{
  "success": true,
  "data": {
    "property_id": "uuid",
    "import_log_id": "uuid",
    "message": "物件を取り込みました"
  }
}
```

**実装例**:
```typescript
// app/api/properties/import/route.ts
import { createRouteHandlerClient } from '@supabase/auth-helpers-nextjs';
import { cookies } from 'next/headers';
import { NextResponse } from 'next/server';
import { openai } from '@ai-sdk/openai';
import { generateObject } from 'ai';
import { z } from 'zod';

const PropertySchema = z.object({
  title: z.string(),
  address: z.string(),
  area: z.string(),
  rent: z.number(),
  layout: z.string(),
  building_type: z.string(),
  floor_area: z.number().optional(),
  station: z.string().optional(),
  distance_from_station: z.number().optional(),
  description: z.string().optional(),
});

export async function POST(request: Request) {
  const { html, url } = await request.json();

  const supabase = createRouteHandlerClient({ cookies });

  // 認証チェック
  const { data: { user }, error: authError } = await supabase.auth.getUser();
  if (authError || !user) {
    return NextResponse.json(
      { success: false, error: { code: 'UNAUTHORIZED', message: '認証が必要です' } },
      { status: 401 }
    );
  }

  // 権限チェック
  const { data: userData } = await supabase
    .from('auth.users')
    .select('role')
    .eq('id', user.id)
    .single();

  if (!['operator', 'admin'].includes(userData.role)) {
    return NextResponse.json(
      { success: false, error: { code: 'FORBIDDEN', message: '権限がありません' } },
      { status: 403 }
    );
  }

  // GPT-4oで構造化データ抽出
  const { object } = await generateObject({
    model: openai('gpt-4o'),
    schema: PropertySchema,
    prompt: `以下のHTMLから物件情報を抽出してください：\n\n${html}`,
  });

  // Embedding生成
  const embeddingResponse = await openai.embeddings.create({
    model: 'text-embedding-3-small',
    input: `${object.title} ${object.description}`,
  });

  // Supabaseに保存
  const { data: property, error: insertError } = await supabase
    .from('properties')
    .insert({
      ...object,
      embedding: embeddingResponse.data[0].embedding,
      created_by: user.id,
    })
    .select()
    .single();

  if (insertError) {
    return NextResponse.json(
      { success: false, error: { code: 'DB_ERROR', message: insertError.message } },
      { status: 500 }
    );
  }

  // HTMLスナップショット保存
  const htmlPath = `html-snapshots/${property.id}.html`;
  await supabase.storage.from('snapshots').upload(htmlPath, html, {
    contentType: 'text/html',
  });

  // 取り込みログ保存
  const { data: importLog } = await supabase
    .from('property_import_logs')
    .insert({
      property_id: property.id,
      source_url: url,
      html_snapshot_path: htmlPath,
      imported_by: user.id,
    })
    .select()
    .single();

  return NextResponse.json({
    success: true,
    data: {
      property_id: property.id,
      import_log_id: importLog.id,
      message: '物件を取り込みました',
    },
  });
}
```

---

### POST /api/properties/search

**説明**: セマンティック検索（pgvector）

**認証**: 必須

**リクエスト**:
```json
{
  "query": "ペット可で広めの2LDK、駅近希望"
}
```

**レスポンス**:
```json
{
  "success": true,
  "data": {
    "properties": [
      {
        "id": "uuid",
        "title": "...",
        "similarity": 0.92
      }
    ]
  }
}
```

**実装例**:
```typescript
// app/api/properties/search/route.ts
export async function POST(request: Request) {
  const { query } = await request.json();

  // クエリのベクトル化
  const embeddingResponse = await openai.embeddings.create({
    model: 'text-embedding-3-small',
    input: query,
  });

  // pgvectorで類似検索
  const { data } = await supabase.rpc('match_properties', {
    query_embedding: embeddingResponse.data[0].embedding,
    match_threshold: 0.8,
    match_count: 10,
  });

  return NextResponse.json({
    success: true,
    data: {
      properties: data,
    },
  });
}
```

---

## 内見予約API ⭐ NEW

### POST /api/viewings

**説明**: 内見予約の作成

**認証**: 必須

**リクエスト**:
```json
{
  "property_id": "uuid",
  "viewing_date": "2025-11-20T14:00:00+09:00",
  "customer_notes": "駐車場の有無を確認したいです"
}
```

**レスポンス**:
```json
{
  "success": true,
  "data": {
    "viewing_id": "uuid",
    "property_id": "uuid",
    "viewing_date": "2025-11-20T14:00:00+09:00",
    "status": "confirmed",
    "message": "内見予約が完了しました。確認メールを送信しました。"
  }
}
```

**実装例**:
```typescript
// app/api/viewings/route.ts
export async function POST(request: Request) {
  const { property_id, viewing_date, customer_notes } = await request.json();

  const supabase = createRouteHandlerClient({ cookies });

  // 認証チェック
  const { data: { user } } = await supabase.auth.getUser();
  if (!user) {
    return NextResponse.json({ error: 'Unauthorized' }, { status: 401 });
  }

  // 顧客情報取得
  const { data: customer } = await supabase
    .from('customers')
    .select('*')
    .eq('user_id', user.id)
    .single();

  // 内見予約作成
  const { data: viewing, error } = await supabase
    .from('viewings')
    .insert({
      customer_id: customer.id,
      property_id,
      viewing_date,
      customer_notes,
      status: 'confirmed',
    })
    .select()
    .single();

  if (error) {
    return NextResponse.json({ error: error.message }, { status: 500 });
  }

  // メール送信（非同期）
  await sendViewingConfirmationEmail(viewing);

  return NextResponse.json({
    success: true,
    data: viewing,
  });
}
```

---

### GET /api/viewings

**説明**: 内見予約一覧取得（自分の予約のみ）

**認証**: 必須

**クエリパラメータ**:
| パラメータ | 型 | 説明 |
|-----------|-----|------|
| status | string | ステータスフィルター |
| from_date | string | 開始日（ISO 8601） |
| to_date | string | 終了日（ISO 8601） |

**レスポンス**:
```json
{
  "success": true,
  "data": {
    "viewings": [
      {
        "id": "uuid",
        "property": {
          "id": "uuid",
          "title": "文京区本郷 2LDK",
          "address": "東京都文京区本郷1-2-3"
        },
        "viewing_date": "2025-11-20T14:00:00+09:00",
        "status": "confirmed",
        "customer_notes": "..."
      }
    ]
  }
}
```

---

### PATCH /api/viewings/[id]

**説明**: 内見予約の更新・キャンセル

**認証**: 必須

**リクエスト**:
```json
{
  "status": "cancelled",
  "cancellation_reason": "都合がつかなくなったため"
}
```

**レスポンス**:
```json
{
  "success": true,
  "data": {
    "viewing_id": "uuid",
    "status": "cancelled",
    "message": "内見予約をキャンセルしました。"
  }
}
```

---

## メール送信API ⭐ NEW

### POST /api/emails/send

**説明**: メール送信（内部API）

**認証**: 必須（operator or admin）

**リクエスト**:
```json
{
  "email_type": "viewing_confirmation",
  "to_email": "customer@example.com",
  "to_name": "山田太郎",
  "template_data": {
    "customer_name": "山田太郎",
    "property_title": "文京区本郷 2LDK",
    "viewing_date": "2025年11月20日 14:00",
    "property_url": "https://ieguchi.jp/properties/uuid"
  },
  "viewing_id": "uuid",
  "include_calendar": true
}
```

**レスポンス**:
```json
{
  "success": true,
  "data": {
    "email_log_id": "uuid",
    "resend_id": "...",
    "status": "sent",
    "sent_at": "2025-11-15T12:00:00Z"
  }
}
```

**実装例**:
```typescript
// app/api/emails/send/route.ts
import { Resend } from 'resend';
import { ViewingConfirmationEmail } from '@/emails/viewing-confirmation';

const resend = new Resend(process.env.RESEND_API_KEY);

export async function POST(request: Request) {
  const {
    email_type,
    to_email,
    to_name,
    template_data,
    viewing_id,
    include_calendar,
  } = await request.json();

  // 認証・権限チェック
  // ...

  let attachments = [];

  // カレンダー招待添付
  if (include_calendar && viewing_id) {
    const { data: viewing } = await supabase
      .from('viewings')
      .select('*, property:properties(*)')
      .eq('id', viewing_id)
      .single();

    const calendar = generateICS(viewing);
    attachments.push({
      filename: 'viewing.ics',
      content: calendar.toString(),
    });
  }

  // メール送信
  const { data, error } = await resend.emails.send({
    from: 'イエグチ <noreply@ieguchi.jp>',
    to: to_email,
    subject: getEmailSubject(email_type),
    react: ViewingConfirmationEmail(template_data),
    attachments,
  });

  if (error) {
    // エラーログ保存
    await supabase.from('email_logs').insert({
      email_type,
      to_email,
      to_name,
      status: 'failed',
      error_message: error.message,
    });

    return NextResponse.json({ error: error.message }, { status: 500 });
  }

  // 送信ログ保存
  const { data: emailLog } = await supabase
    .from('email_logs')
    .insert({
      email_type,
      to_email,
      to_name,
      subject: getEmailSubject(email_type),
      template_name: 'viewing_confirmation',
      viewing_id,
      status: 'sent',
      resend_id: data.id,
      sent_at: new Date().toISOString(),
    })
    .select()
    .single();

  return NextResponse.json({
    success: true,
    data: emailLog,
  });
}
```

---

### GET /api/emails/logs

**説明**: メール送信ログ取得（管理者のみ）

**認証**: 必須（admin）

**クエリパラメータ**:
| パラメータ | 型 | 説明 |
|-----------|-----|------|
| email_type | string | メール種別 |
| status | string | 送信状態 |
| from_date | string | 開始日 |
| to_date | string | 終了日 |
| limit | number | 取得件数 |

**レスポンス**:
```json
{
  "success": true,
  "data": {
    "email_logs": [
      {
        "id": "uuid",
        "email_type": "viewing_confirmation",
        "to_email": "customer@example.com",
        "subject": "内見予約が確定しました",
        "status": "sent",
        "sent_at": "2025-11-15T12:00:00Z"
      }
    ],
    "pagination": {
      "total": 150,
      "page": 1,
      "limit": 20
    }
  }
}
```

---

## 管理者API

### GET /api/admin/users

**説明**: ユーザー一覧取得（管理者のみ）

**認証**: 必須（admin）

**レスポンス**:
```json
{
  "success": true,
  "data": {
    "users": [
      {
        "id": "uuid",
        "email": "user@example.com",
        "role": "customer",
        "created_at": "2025-01-01T00:00:00Z"
      }
    ]
  }
}
```

---

### GET /api/admin/logs

**説明**: 取り込みログ一覧（管理者のみ）

**認証**: 必須（admin）

**レスポンス**:
```json
{
  "success": true,
  "data": {
    "logs": [
      {
        "id": "uuid",
        "property_id": "uuid",
        "source_url": "https://...",
        "imported_by": "uuid",
        "imported_at": "2025-01-01T00:00:00Z"
      }
    ]
  }
}
```

---

## レート制限

### 実装（Upstash Redis）

```typescript
// middleware.ts
import { Ratelimit } from '@upstash/ratelimit';
import { kv } from '@vercel/kv';

const ratelimit = new Ratelimit({
  redis: kv,
  limiter: Ratelimit.slidingWindow(10, '10 s'), // 10秒で10リクエスト
});

export async function middleware(request: NextRequest) {
  const ip = request.headers.get('x-forwarded-for');

  const { success } = await ratelimit.limit(ip);

  if (!success) {
    return new Response('Too Many Requests', { status: 429 });
  }

  return NextResponse.next();
}
```

---

## エラーコード一覧

| コード | 説明 |
|--------|------|
| AUTH_FAILED | 認証失敗 |
| UNAUTHORIZED | 認証が必要 |
| FORBIDDEN | 権限不足 |
| NOT_FOUND | リソースが見つからない |
| VALIDATION_ERROR | バリデーションエラー |
| DB_ERROR | データベースエラー |
| OPENAI_ERROR | OpenAI APIエラー |
| ELEVENLABS_ERROR | ElevenLabs APIエラー |
| RATE_LIMIT_EXCEEDED | レート制限超過 |
| INTERNAL_ERROR | 内部サーバーエラー |

---

次のドキュメント: [コスト見積もり](./cost-estimation.md)
