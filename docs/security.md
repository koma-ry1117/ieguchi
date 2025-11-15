# セキュリティ設計

## セキュリティ方針

**多層防御（Defense in Depth）** の原則に基づき、複数のセキュリティ層を組み合わせて保護します。

```
┌─────────────────────────────────────┐
│   1. ネットワーク層セキュリティ      │ ← HTTPS, CORS, CSP
├─────────────────────────────────────┤
│   2. 認証・認可層                   │ ← Supabase Auth, JWT, RLS
├─────────────────────────────────────┤
│   3. アプリケーション層              │ ← 入力検証, レート制限
├─────────────────────────────────────┤
│   4. データベース層                  │ ← RLS, 暗号化
├─────────────────────────────────────┤
│   5. 監視・監査層                    │ ← ログ記録, 異常検知
└─────────────────────────────────────┘
```

---

## 1. 認証・認可

### 認証方式

#### Supabase Auth（JWT ベース）

**特徴:**
- JWT（JSON Web Token）によるステートレス認証
- Access Token（短期有効）+ Refresh Token（長期有効）
- セッション自動更新
- OAuth 2.0対応

**実装:**

```typescript
// ログイン
const { data, error } = await supabase.auth.signInWithPassword({
  email: 'user@example.com',
  password: 'password',
});

// セッション確認
const { data: { user } } = await supabase.auth.getUser();

// ログアウト
await supabase.auth.signOut();
```

**トークン仕様:**
- Access Token 有効期限: 1時間
- Refresh Token 有効期限: 30日
- 自動リフレッシュ機能あり

---

### 認可（ロールベースアクセス制御）

#### ユーザーロール

| ロール | 権限 |
|--------|------|
| customer | 物件閲覧、会話履歴閲覧（自分のみ） |
| operator | 物件取り込み、自分の物件編集、顧客対応履歴確認 |
| admin | 全物件編集・削除、ユーザー管理、システム設定、ログ確認 |

#### Row Level Security (RLS)

**データベースレベルでのアクセス制御**

```sql
-- properties テーブル: 顧客は公開物件のみ閲覧可能
CREATE POLICY "customer_view_public"
ON properties FOR SELECT
TO authenticated
USING (
  is_public = true
  AND (SELECT role FROM auth.users WHERE id = auth.uid()) = 'customer'
);

-- properties テーブル: オペレーターは自分が登録した物件のみ更新可能
CREATE POLICY "operator_update_own"
ON properties FOR UPDATE
TO authenticated
USING (
  created_by = auth.uid()
  AND (SELECT role FROM auth.users WHERE id = auth.uid()) = 'operator'
);

-- properties テーブル: 管理者はすべての操作が可能
CREATE POLICY "admin_all"
ON properties FOR ALL
TO authenticated
USING (
  (SELECT role FROM auth.users WHERE id = auth.uid()) = 'admin'
);
```

**メリット:**
- アプリケーションコードをバイパスしてもデータ保護される
- SQLレベルでの強力なアクセス制御
- ロール変更時の自動反映

---

### セッション管理

#### セッション有効期限

```typescript
// middleware.ts
import { createMiddlewareClient } from '@supabase/auth-helpers-nextjs';

export async function middleware(request: NextRequest) {
  const res = NextResponse.next();
  const supabase = createMiddlewareClient({ req: request, res });

  // セッション確認・自動更新
  const { data: { session } } = await supabase.auth.getSession();

  if (!session) {
    return NextResponse.redirect(new URL('/login', request.url));
  }

  return res;
}
```

#### セッション無効化

```typescript
// 強制ログアウト（管理者機能）
await supabase.auth.admin.signOut(userId);

// 全セッション無効化
await supabase.auth.admin.deleteUser(userId);
```

---

## 2. データ保護

### データ暗号化

#### 保存時の暗号化（Encryption at Rest）

**Supabase（PostgreSQL）**
- AES-256による自動暗号化
- バックアップデータも暗号化

**機密データの追加暗号化**

```typescript
import crypto from 'crypto';

const ENCRYPTION_KEY = process.env.ENCRYPTION_KEY; // 32バイト

function encrypt(text: string): string {
  const iv = crypto.randomBytes(16);
  const cipher = crypto.createCipheriv('aes-256-cbc', Buffer.from(ENCRYPTION_KEY), iv);

  let encrypted = cipher.update(text, 'utf8', 'hex');
  encrypted += cipher.final('hex');

  return iv.toString('hex') + ':' + encrypted;
}

function decrypt(text: string): string {
  const parts = text.split(':');
  const iv = Buffer.from(parts[0], 'hex');
  const encryptedText = parts[1];

  const decipher = crypto.createDecipheriv('aes-256-cbc', Buffer.from(ENCRYPTION_KEY), iv);

  let decrypted = decipher.update(encryptedText, 'hex', 'utf8');
  decrypted += decipher.final('utf8');

  return decrypted;
}

// 使用例
const encryptedPhone = encrypt(customer.phone);
await supabase.from('customers').insert({ phone: encryptedPhone });
```

#### 通信時の暗号化（Encryption in Transit）

- **HTTPS強制**: すべての通信をTLS 1.3で暗号化
- **HSTS**: HTTP Strict Transport Security有効化

```typescript
// next.config.js
module.exports = {
  async headers() {
    return [
      {
        source: '/:path*',
        headers: [
          {
            key: 'Strict-Transport-Security',
            value: 'max-age=63072000; includeSubDomains; preload',
          },
        ],
      },
    ];
  },
};
```

---

### パスワード管理

#### パスワードポリシー

**要件:**
- 最低8文字以上
- 大文字・小文字・数字・記号を含む
- よくあるパスワードは禁止

**実装（Zod）:**

```typescript
import { z } from 'zod';

const passwordSchema = z
  .string()
  .min(8, 'パスワードは8文字以上必要です')
  .regex(/[A-Z]/, '大文字を含める必要があります')
  .regex(/[a-z]/, '小文字を含める必要があります')
  .regex(/[0-9]/, '数字を含める必要があります')
  .regex(/[^A-Za-z0-9]/, '記号を含める必要があります');
```

#### パスワードハッシュ化

Supabase Authが自動的に**bcrypt**でハッシュ化

---

## 3. 入力検証・XSS対策

### 入力検証

#### Zodスキーマ検証

```typescript
import { z } from 'zod';

const propertySearchSchema = z.object({
  area: z.string().max(100).optional(),
  rent_min: z.number().min(0).max(999999999).optional(),
  rent_max: z.number().min(0).max(999999999).optional(),
  layout: z.string().max(50).optional(),
});

// API Routeでの使用
export async function POST(request: Request) {
  const body = await request.json();

  // バリデーション
  const result = propertySearchSchema.safeParse(body);

  if (!result.success) {
    return NextResponse.json(
      { error: { code: 'VALIDATION_ERROR', details: result.error.errors } },
      { status: 400 }
    );
  }

  // 処理続行
  const validatedData = result.data;
  // ...
}
```

### XSS（クロスサイトスクリプティング）対策

#### React自動エスケープ

Reactは**デフォルトでHTML自動エスケープ**

```jsx
// 安全（自動エスケープ）
<div>{userInput}</div>

// 危険（使用しない）
<div dangerouslySetInnerHTML={{ __html: userInput }} />
```

#### DOMPurifyによるサニタイゼーション

```typescript
import DOMPurify from 'isomorphic-dompurify';

const sanitizedHTML = DOMPurify.sanitize(userInput);
```

#### Content Security Policy (CSP)

```typescript
// middleware.ts
export function middleware(request: NextRequest) {
  const response = NextResponse.next();

  response.headers.set(
    'Content-Security-Policy',
    [
      "default-src 'self'",
      "script-src 'self' 'unsafe-eval' 'unsafe-inline' https://vercel.live",
      "style-src 'self' 'unsafe-inline'",
      "img-src 'self' data: https:",
      "font-src 'self' data:",
      "connect-src 'self' https://*.supabase.co wss://*.supabase.co https://api.openai.com https://api.elevenlabs.io",
      "media-src 'self' https:",
    ].join('; ')
  );

  return response;
}
```

---

## 4. SQLインジェクション対策

### Supabaseクライアント使用

Supabase JavaScriptクライアントは**自動的にパラメータ化クエリ**を使用

```typescript
// 安全
const { data } = await supabase
  .from('properties')
  .select('*')
  .eq('area', userInput); // 自動エスケープ

// 危険（使用しない）
await supabase.rpc('raw_sql', { query: `SELECT * FROM properties WHERE area = '${userInput}'` });
```

### プリペアドステートメント

```sql
-- 安全なストアドプロシージャ
CREATE OR REPLACE FUNCTION search_properties_safe(
  p_area TEXT,
  p_rent_max INTEGER
)
RETURNS TABLE (...) AS $$
BEGIN
  RETURN QUERY
  SELECT * FROM properties
  WHERE area = p_area
    AND rent <= p_rent_max;
END;
$$ LANGUAGE plpgsql;
```

---

## 5. CSRF（クロスサイトリクエストフォージェリ）対策

### トークンベース認証

JWT認証により、Cookie利用時のCSRF攻撃を軽減

### SameSite Cookie属性

```typescript
// Supabase Authのセッション管理で自動設定
// Set-Cookie: sb-access-token=...; SameSite=Lax; Secure; HttpOnly
```

---

## 6. レート制限

### Upstash Redis + Vercel Edge Middleware

```typescript
// middleware.ts
import { Ratelimit } from '@upstash/ratelimit';
import { kv } from '@vercel/kv';

const ratelimit = new Ratelimit({
  redis: kv,
  limiter: Ratelimit.slidingWindow(10, '10 s'), // 10秒で10リクエスト
  analytics: true,
});

export async function middleware(request: NextRequest) {
  const ip = request.headers.get('x-forwarded-for') || 'unknown';

  const { success, limit, remaining } = await ratelimit.limit(ip);

  const response = success ? NextResponse.next() : new Response('Too Many Requests', { status: 429 });

  response.headers.set('X-RateLimit-Limit', limit.toString());
  response.headers.set('X-RateLimit-Remaining', remaining.toString());

  return response;
}

export const config = {
  matcher: '/api/:path*',
};
```

### エンドポイント別のレート制限

```typescript
// app/api/chat/route.ts
const chatRatelimit = new Ratelimit({
  redis: kv,
  limiter: Ratelimit.slidingWindow(5, '60 s'), // 1分間に5回
});

export async function POST(request: Request) {
  const ip = request.headers.get('x-forwarded-for');
  const { success } = await chatRatelimit.limit(ip);

  if (!success) {
    return new Response('Too Many Requests', { status: 429 });
  }

  // 処理続行
}
```

---

## 7. セキュリティヘッダー

### Next.js設定

```typescript
// next.config.js
module.exports = {
  async headers() {
    return [
      {
        source: '/:path*',
        headers: [
          // XSS対策
          {
            key: 'X-Content-Type-Options',
            value: 'nosniff',
          },
          // クリックジャッキング対策
          {
            key: 'X-Frame-Options',
            value: 'DENY',
          },
          // HTTPS強制
          {
            key: 'Strict-Transport-Security',
            value: 'max-age=63072000; includeSubDomains; preload',
          },
          // リファラー制御
          {
            key: 'Referrer-Policy',
            value: 'strict-origin-when-cross-origin',
          },
          // 権限ポリシー
          {
            key: 'Permissions-Policy',
            value: 'camera=(), microphone=(self), geolocation=()',
          },
        ],
      },
    ];
  },
};
```

---

## 8. CORS設定

### Next.js API Routes

```typescript
// app/api/chat/route.ts
export async function POST(request: Request) {
  // CORS設定
  const allowedOrigins = [
    'https://ieguchi.vercel.app',
    'http://localhost:3000',
  ];

  const origin = request.headers.get('origin');

  if (!allowedOrigins.includes(origin)) {
    return new Response('Forbidden', { status: 403 });
  }

  const response = NextResponse.json({ ... });

  response.headers.set('Access-Control-Allow-Origin', origin);
  response.headers.set('Access-Control-Allow-Methods', 'POST, OPTIONS');
  response.headers.set('Access-Control-Allow-Headers', 'Content-Type, Authorization');

  return response;
}

// OPTIONSメソッド対応（プリフライト）
export async function OPTIONS(request: Request) {
  const origin = request.headers.get('origin');

  return new Response(null, {
    status: 204,
    headers: {
      'Access-Control-Allow-Origin': origin,
      'Access-Control-Allow-Methods': 'POST, OPTIONS',
      'Access-Control-Allow-Headers': 'Content-Type, Authorization',
    },
  });
}
```

---

## 9. API Key管理

### 環境変数による管理

```bash
# .env.local
OPENAI_API_KEY=sk-...
ELEVENLABS_API_KEY=...
SUPABASE_SERVICE_ROLE_KEY=...
NEXT_PUBLIC_SUPABASE_URL=...
NEXT_PUBLIC_SUPABASE_ANON_KEY=...
```

**重要:**
- `NEXT_PUBLIC_` プレフィックスのない変数はサーバーサイドのみ
- Gitに `.env.local` をコミットしない（`.gitignore` に追加）
- Vercelの環境変数設定で管理

### Vercel環境変数設定

```bash
# Vercel CLIでの設定
vercel env add OPENAI_API_KEY

# または Vercel Dashboardで設定
# Project Settings > Environment Variables
```

---

## 10. ログ記録・監査

### アクセスログ

```typescript
// lib/logger.ts
import { Logger } from 'next-axiom';

export async function logAccess(userId: string, action: string, details: any) {
  const logger = new Logger();

  logger.info('Access Log', {
    userId,
    action,
    details,
    timestamp: new Date().toISOString(),
  });

  await logger.flush();
}

// 使用例
await logAccess(user.id, 'property_import', {
  propertyId: property.id,
  sourceUrl: url,
});
```

### セキュリティイベントログ

```typescript
// 不正アクセス試行の記録
export async function logSecurityEvent(event: string, details: any) {
  await supabase.from('security_logs').insert({
    event,
    details,
    ip_address: request.headers.get('x-forwarded-for'),
    user_agent: request.headers.get('user-agent'),
    timestamp: new Date().toISOString(),
  });
}

// 使用例
await logSecurityEvent('unauthorized_access_attempt', {
  endpoint: '/api/admin/users',
  userId: user.id,
  role: user.role,
});
```

### 監査ログ（Audit Trail）

```sql
-- property_import_logs テーブル（すでに定義済み）
CREATE TABLE property_import_logs (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  property_id UUID REFERENCES properties(id),
  source_url TEXT NOT NULL,
  html_snapshot_path TEXT,
  imported_by UUID NOT NULL REFERENCES auth.users(id),
  imported_at TIMESTAMPTZ DEFAULT NOW(),
  import_method VARCHAR(50) DEFAULT 'chrome_extension',
  status VARCHAR(20) DEFAULT 'success'
);
```

**用途**: 法的証跡・監査対応

---

## 11. 脆弱性スキャン

### 依存関係の脆弱性チェック

```bash
# npm audit
npm audit

# pnpm audit
pnpm audit

# 自動修正
pnpm audit --fix
```

### Dependabot（GitHub）

```yaml
# .github/dependabot.yml
version: 2
updates:
  - package-ecosystem: "npm"
    directory: "/"
    schedule:
      interval: "weekly"
    open-pull-requests-limit: 10
```

### Snyk統合

```bash
# Snykインストール
npm install -g snyk

# スキャン
snyk test

# 監視
snyk monitor
```

---

## 12. Chrome拡張機能のセキュリティ

### Manifest V3 セキュリティ強化

```json
{
  "manifest_version": 3,
  "permissions": [
    "activeTab",
    "storage"
  ],
  "host_permissions": [
    "https://www.athome.co.jp/*"
  ],
  "content_security_policy": {
    "extension_pages": "script-src 'self'; object-src 'self'"
  }
}
```

### Auth Token安全管理

```typescript
// background.ts
import { storage } from 'webextension-polyfill';

// トークン保存（暗号化）
await storage.local.set({
  authToken: encryptToken(token),
});

// トークン取得（復号化）
const { authToken } = await storage.local.get('authToken');
const token = decryptToken(authToken);
```

### 通信セキュリティ

```typescript
// HTTPSのみ許可
const API_BASE_URL = 'https://ieguchi.vercel.app/api';

// Authorization Header使用
fetch(`${API_BASE_URL}/properties/import`, {
  method: 'POST',
  headers: {
    'Content-Type': 'application/json',
    'Authorization': `Bearer ${token}`,
  },
  body: JSON.stringify({ html, url }),
});
```

---

## 13. インシデント対応計画

### セキュリティインシデント対応フロー

```
1. 検知 → 2. 初動対応 → 3. 調査 → 4. 封じ込め → 5. 復旧 → 6. 事後分析
```

#### 1. 検知
- Sentryアラート
- 異常なアクセスログ
- ユーザーからの報告

#### 2. 初動対応（30分以内）
- インシデント対応チーム招集
- 影響範囲の特定
- 緊急連絡

#### 3. 調査
- ログ分析
- 原因特定
- 影響範囲の確定

#### 4. 封じ込め
- 脆弱性のある機能の一時停止
- 不正アクセスのブロック
- セッションの強制無効化

#### 5. 復旧
- パッチ適用
- システム再起動
- 動作確認

#### 6. 事後分析
- 根本原因分析
- 再発防止策の策定
- ドキュメント更新

---

## 14. セキュリティチェックリスト

### 開発時

- [ ] 環境変数に機密情報を保存
- [ ] `.env.local` をGitにコミットしない
- [ ] Zodでバリデーション実装
- [ ] React自動エスケープを活用
- [ ] Supabaseクライアント使用（SQLインジェクション対策）

### デプロイ前

- [ ] HTTPS強制設定
- [ ] セキュリティヘッダー設定
- [ ] CORS設定
- [ ] レート制限実装
- [ ] CSP設定
- [ ] 依存関係の脆弱性スキャン

### 運用時

- [ ] アクセスログ監視
- [ ] セキュリティイベント監視
- [ ] 定期的な脆弱性スキャン
- [ ] 定期的なバックアップ
- [ ] インシデント対応訓練

---

## 15. コンプライアンス

### 個人情報保護法対応

**収集する個人情報:**
- 氏名
- メールアドレス
- 電話番号
- 会話履歴

**対策:**
- プライバシーポリシーの明示
- 利用目的の明示
- データの適切な管理・暗号化
- データ削除要求への対応

### GDPR対応（将来的な海外展開時）

- ユーザーの同意取得
- データポータビリティ
- 削除権（Right to be Forgotten）
- データ処理の透明性

---

## まとめ

このセキュリティ設計により、以下を実現します:

1. **多層防御**: ネットワーク〜データベースまで複数のセキュリティ層
2. **強固な認証**: JWT + RLSによる堅牢な認証・認可
3. **データ保護**: 暗号化・バリデーション・エスケープ
4. **攻撃対策**: XSS, SQLインジェクション, CSRF, レート制限
5. **監視・監査**: ログ記録・異常検知・インシデント対応

**定期的なセキュリティレビュー**を実施し、最新の脅威に対応していきます。

---

[トップページに戻る](../README.md)
