# データベース設計

## データベース構成

Supabase PostgreSQL + pgvector を使用

---

## ER図

```
┌─────────────────┐
│   auth.users    │ (Supabase管理)
│─────────────────│
│ id (uuid) PK    │
│ email           │
│ role            │ ← 'admin', 'operator', 'customer'
│ created_at      │
└────────┬────────┘
         │
         │ 1:1 (role='operator'の場合)
         │
┌────────▼──────────────────┐
│      operators            │ ← ⭐ NEW オペレーター企業テーブル
│───────────────────────────│
│ id (uuid) PK              │
│ user_id (uuid) FK         │ ← auth.users.id (role='operator')
│ company_name              │
│ company_address           │
│ phone                     │
│ email                     │
│ subscription_status       │ ← 'active', 'suspended', 'cancelled'
│ subscription_plan         │ ← 'basic', 'standard', 'premium'
│ contract_start_date       │
│ contract_end_date         │
│ is_active                 │
│ created_at                │
│ updated_at                │
└────────┬──────────────────┘
         │
         │ 1:N
         │
┌────────▼────────────────────────┐
│      properties                 │
│─────────────────────────────────│
│ id (uuid) PK                    │
│ operator_id (uuid) FK           │ ← ⭐ NEW どのオペレーターの物件か
│ title                           │
│ address                         │
│ area                            │
│ rent                            │
│ layout                          │
│ building_type                   │
│ floor_area                      │
│ station                         │
│ distance_from_station           │
│ description                     │
│ embedding (vector)              │ ← pgvector
│ is_public                       │
│ created_by (uuid) FK            │ ← auth.users.id
│ created_at                      │
│ updated_at                      │
└────────┬────────────────────────┘
         │
         │ 1:N
         │
┌────────▼──────────────┐
│  property_images      │
│───────────────────────│
│ id (uuid) PK          │
│ property_id (uuid) FK │
│ storage_path          │
│ display_order         │
│ created_at            │
└───────────────────────┘

┌──────────────────────────┐
│ property_import_logs     │
│──────────────────────────│
│ id (uuid) PK             │
│ property_id (uuid) FK    │
│ operator_id (uuid) FK    │ ← ⭐ NEW
│ source_url               │
│ html_snapshot_path       │
│ imported_by (uuid) FK    │
│ imported_at              │
└──────────────────────────┘

┌──────────────────┐
│   customers      │
│──────────────────│
│ id (uuid) PK     │
│ user_id (uuid) FK│ ← auth.users.id
│ operator_id (FK) │ ← ⭐ NEW どのオペレーターの顧客か
│ name             │
│ phone            │
│ email            │
│ created_at       │
└────────┬─────────┘
         │
         │ 1:N
         │
┌────────▼──────────────┐
│ customer_preferences  │
│───────────────────────│
│ id (uuid) PK          │
│ customer_id (uuid) FK │
│ area                  │
│ rent_min              │
│ rent_max              │
│ layout                │
│ building_type         │
│ updated_at            │
└───────────────────────┘

┌──────────────────────┐
│   conversations      │
│──────────────────────│
│ id (uuid) PK         │
│ customer_id (uuid) FK│
│ operator_id (uuid) FK│ ← ⭐ NEW
│ started_at           │
│ ended_at             │
└────────┬─────────────┘
         │
         │ 1:N
         │
┌────────▼─────────┐
│    messages      │
│──────────────────│
│ id (uuid) PK     │
│ conversation_id  │
│ role             │
│ content          │
│ emotion          │
│ timestamp        │
└──────────────────┘

┌──────────────────┐
│   inquiries      │
│──────────────────│
│ id (uuid) PK     │
│ customer_id (FK) │
│ property_id (FK) │
│ operator_id (FK) │ ← ⭐ NEW
│ status           │
│ message          │
│ created_at       │
└──────────────────┘

┌──────────────────┐
│   viewings       │
│──────────────────│
│ id (uuid) PK     │
│ customer_id (FK) │
│ property_id (FK) │
│ operator_id (FK) │ ← ⭐ NEW
│ viewing_date     │
│ status           │
│ created_at       │
└──────────────────┘
```

---

## テーブル定義

### 1. operators（オペレーター企業マスタ） ⭐ NEW

```sql
CREATE TABLE operators (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),

  -- ユーザーアカウント紐付け
  user_id UUID UNIQUE NOT NULL REFERENCES auth.users(id) ON DELETE CASCADE,

  -- 企業情報
  company_name VARCHAR(255) NOT NULL,
  company_address TEXT,
  phone VARCHAR(20),
  email VARCHAR(255) NOT NULL,

  -- サブスクリプション情報
  subscription_status VARCHAR(20) DEFAULT 'active' CHECK (
    subscription_status IN ('active', 'suspended', 'cancelled', 'trial')
  ),
  subscription_plan VARCHAR(20) DEFAULT 'basic' CHECK (
    subscription_plan IN ('basic', 'standard', 'premium')
  ),

  -- 契約情報
  contract_start_date DATE NOT NULL,
  contract_end_date DATE,

  -- ステータス
  is_active BOOLEAN DEFAULT true,

  -- メタ情報
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW()
);

-- インデックス作成
CREATE INDEX idx_operators_user_id ON operators(user_id);
CREATE INDEX idx_operators_subscription_status ON operators(subscription_status);
CREATE INDEX idx_operators_is_active ON operators(is_active);

-- 更新日時の自動更新
CREATE TRIGGER update_operators_updated_at
BEFORE UPDATE ON operators
FOR EACH ROW
EXECUTE FUNCTION update_updated_at_column();
```

**カラム説明**:
| カラム名 | 型 | 制約 | 説明 |
|---------|-----|------|------|
| id | uuid | PK | オペレーターID |
| user_id | uuid | FK, UNIQUE | auth.usersへの参照 |
| company_name | varchar(255) | NOT NULL | 企業名 |
| company_address | text | | 企業住所 |
| phone | varchar(20) | | 電話番号 |
| email | varchar(255) | NOT NULL | 企業メールアドレス |
| subscription_status | varchar(20) | NOT NULL | サブスク状態 |
| subscription_plan | varchar(20) | NOT NULL | プラン種別 |
| contract_start_date | date | NOT NULL | 契約開始日 |
| contract_end_date | date | | 契約終了日 |
| is_active | boolean | DEFAULT true | 有効フラグ |
| created_at | timestamptz | DEFAULT NOW() | 作成日時 |
| updated_at | timestamptz | DEFAULT NOW() | 更新日時 |

---

### 2. auth.users（Supabase管理テーブル）

```sql
-- Supabaseが自動生成・管理
-- カスタムフィールドを追加

ALTER TABLE auth.users
ADD COLUMN role VARCHAR(20) DEFAULT 'customer';

-- roleの制約
ALTER TABLE auth.users
ADD CONSTRAINT check_role
CHECK (role IN ('customer', 'operator', 'admin'));
```

**カラム説明**:
| カラム名 | 型 | 制約 | 説明 |
|---------|-----|------|------|
| id | uuid | PK | ユーザーID（Supabase自動生成） |
| email | varchar | UNIQUE | メールアドレス |
| role | varchar(20) | NOT NULL | ロール（customer/operator/admin） |
| created_at | timestamptz | NOT NULL | 作成日時 |

---

### 3. properties（物件マスタ）

```sql
CREATE TABLE properties (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),

  -- オペレーター紐付け ⭐ NEW
  operator_id UUID NOT NULL REFERENCES operators(id) ON DELETE CASCADE,

  -- 基本情報
  title VARCHAR(255) NOT NULL,
  address TEXT NOT NULL,
  area VARCHAR(100) NOT NULL, -- 文京区、千代田区など

  -- 賃料・売買
  transaction_type VARCHAR(20) NOT NULL CHECK (transaction_type IN ('rent', 'sale')),
  rent INTEGER, -- 賃料（円/月） ※賃貸の場合
  sale_price BIGINT, -- 売買価格（円） ※売買の場合
  management_fee INTEGER, -- 管理費（円/月）
  key_money INTEGER, -- 礼金（ヶ月）
  deposit INTEGER, -- 敷金（ヶ月）

  -- 物件詳細
  layout VARCHAR(50) NOT NULL, -- 1K、1LDK、2LDKなど
  building_type VARCHAR(50) NOT NULL, -- マンション、一戸建て、アパート
  floor_area DECIMAL(6,2), -- 専有面積（㎡）
  floor_number INTEGER, -- 階数
  total_floors INTEGER, -- 総階数
  year_built INTEGER, -- 築年
  structure VARCHAR(50), -- 構造（RC造、木造など）

  -- 設備
  amenities JSONB, -- {balcony: true, parking: true, pet: false, ...}

  -- アクセス
  station VARCHAR(100), -- 最寄り駅
  distance_from_station INTEGER, -- 駅徒歩（分）

  -- 説明文
  description TEXT,

  -- ベクトル検索用（pgvector）
  embedding vector(1536), -- OpenAI Embeddings

  -- 公開設定
  is_public BOOLEAN DEFAULT true,

  -- メタ情報
  created_by UUID REFERENCES auth.users(id),
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW()
);

-- インデックス作成
CREATE INDEX idx_properties_operator_id ON properties(operator_id);
CREATE INDEX idx_properties_area ON properties(area);
CREATE INDEX idx_properties_rent ON properties(rent);
CREATE INDEX idx_properties_layout ON properties(layout);
CREATE INDEX idx_properties_transaction_type ON properties(transaction_type);

-- 複合インデックス
CREATE INDEX idx_properties_search
ON properties(area, rent, layout, operator_id)
WHERE is_public = true;

-- ベクトル検索用インデックス（pgvector）
CREATE INDEX idx_properties_embedding
ON properties
USING ivfflat (embedding vector_cosine_ops)
WITH (lists = 100);

-- 全文検索用インデックス
CREATE INDEX idx_properties_fulltext
ON properties
USING gin(to_tsvector('japanese', title || ' ' || description));

-- 更新日時の自動更新
CREATE TRIGGER update_properties_updated_at
BEFORE UPDATE ON properties
FOR EACH ROW
EXECUTE FUNCTION update_updated_at_column();
```

**カラム説明**:
| カラム名 | 型 | 制約 | 説明 |
|---------|-----|------|------|
| id | uuid | PK | 物件ID |
| operator_id | uuid | FK, NOT NULL | オペレーターID（どの不動産会社の物件か） |
| title | varchar(255) | NOT NULL | 物件タイトル |
| address | text | NOT NULL | 住所 |
| area | varchar(100) | NOT NULL | エリア（文京区等） |
| transaction_type | varchar(20) | NOT NULL | 取引種別（rent/sale） |
| rent | integer | | 賃料（円/月） |
| sale_price | bigint | | 売買価格（円） |
| layout | varchar(50) | NOT NULL | 間取り |
| building_type | varchar(50) | NOT NULL | 建物種別 |
| floor_area | decimal(6,2) | | 専有面積（㎡） |
| station | varchar(100) | | 最寄り駅 |
| distance_from_station | integer | | 駅徒歩（分） |
| description | text | | 物件説明 |
| embedding | vector(1536) | | ベクトル（セマンティック検索用） |
| is_public | boolean | DEFAULT true | 公開フラグ |
| created_by | uuid | FK | 登録者（auth.users.id） |
| created_at | timestamptz | DEFAULT NOW() | 作成日時 |
| updated_at | timestamptz | DEFAULT NOW() | 更新日時 |

---

### 3. property_images（物件画像）

```sql
CREATE TABLE property_images (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  property_id UUID NOT NULL REFERENCES properties(id) ON DELETE CASCADE,
  storage_path TEXT NOT NULL, -- Supabase Storageのパス
  display_order INTEGER NOT NULL DEFAULT 0,
  created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_property_images_property_id ON property_images(property_id);
CREATE INDEX idx_property_images_display_order ON property_images(property_id, display_order);
```

---

### 4. property_import_logs（取り込みログ）

```sql
CREATE TABLE property_import_logs (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  property_id UUID REFERENCES properties(id) ON DELETE SET NULL,
  operator_id UUID NOT NULL REFERENCES operators(id) ON DELETE CASCADE, -- ⭐ NEW
  source_url TEXT NOT NULL, -- 元のURL（アットホーム等）
  html_snapshot_path TEXT, -- Supabase Storageに保存したHTMLパス
  imported_by UUID NOT NULL REFERENCES auth.users(id),
  imported_at TIMESTAMPTZ DEFAULT NOW(),

  -- メタ情報
  import_method VARCHAR(50) DEFAULT 'chrome_extension', -- インポート方法
  status VARCHAR(20) DEFAULT 'success' CHECK (status IN ('success', 'failed', 'pending')),
  error_message TEXT
);

CREATE INDEX idx_import_logs_operator_id ON property_import_logs(operator_id);
CREATE INDEX idx_import_logs_imported_by ON property_import_logs(imported_by);
CREATE INDEX idx_import_logs_imported_at ON property_import_logs(imported_at DESC);
```

**用途**: 法的証跡・監査ログ

---

### 5. customers（顧客情報）

```sql
CREATE TABLE customers (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id UUID UNIQUE REFERENCES auth.users(id) ON DELETE CASCADE,
  operator_id UUID NOT NULL REFERENCES operators(id) ON DELETE CASCADE, -- ⭐ NEW どのオペレーターの顧客か
  name VARCHAR(100),
  phone VARCHAR(20),
  email VARCHAR(255),
  created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_customers_user_id ON customers(user_id);
CREATE INDEX idx_customers_operator_id ON customers(operator_id);
```

---

### 6. customer_preferences（顧客の希望条件）

```sql
CREATE TABLE customer_preferences (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  customer_id UUID NOT NULL REFERENCES customers(id) ON DELETE CASCADE,

  -- 希望条件
  area VARCHAR(100),
  rent_min INTEGER,
  rent_max INTEGER,
  layout VARCHAR(50),
  building_type VARCHAR(50),
  station VARCHAR(100),
  max_distance_from_station INTEGER, -- 駅徒歩上限（分）

  -- JSONB形式での詳細条件
  conditions JSONB, -- {balcony: true, pet_allowed: true, ...}

  updated_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_customer_preferences_customer_id ON customer_preferences(customer_id);
```

---

### 7. conversations（会話セッション）

```sql
CREATE TABLE conversations (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  customer_id UUID REFERENCES customers(id) ON DELETE SET NULL,
  operator_id UUID NOT NULL REFERENCES operators(id) ON DELETE CASCADE, -- ⭐ NEW
  started_at TIMESTAMPTZ DEFAULT NOW(),
  ended_at TIMESTAMPTZ,

  -- 会話のメタ情報
  total_messages INTEGER DEFAULT 0,
  satisfaction_score INTEGER CHECK (satisfaction_score BETWEEN 1 AND 5)
);

CREATE INDEX idx_conversations_customer_id ON conversations(customer_id);
CREATE INDEX idx_conversations_operator_id ON conversations(operator_id);
CREATE INDEX idx_conversations_started_at ON conversations(started_at DESC);
```

---

### 8. messages（会話履歴）

```sql
CREATE TABLE messages (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  conversation_id UUID NOT NULL REFERENCES conversations(id) ON DELETE CASCADE,

  role VARCHAR(20) NOT NULL CHECK (role IN ('user', 'assistant', 'system')),
  content TEXT NOT NULL,

  -- アバター表情（assistantの場合のみ）
  emotion VARCHAR(20) CHECK (emotion IN ('neutral', 'happy', 'thinking', 'apologetic', 'excited')),

  timestamp TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_messages_conversation_id ON messages(conversation_id);
CREATE INDEX idx_messages_timestamp ON messages(conversation_id, timestamp);
```

---

### 9. inquiries（問い合わせ）

```sql
CREATE TABLE inquiries (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  customer_id UUID REFERENCES customers(id) ON DELETE SET NULL,
  property_id UUID REFERENCES properties(id) ON DELETE SET NULL,
  operator_id UUID NOT NULL REFERENCES operators(id) ON DELETE CASCADE, -- ⭐ NEW

  status VARCHAR(20) DEFAULT 'pending' CHECK (status IN ('pending', 'in_progress', 'completed', 'cancelled')),

  message TEXT NOT NULL,
  response TEXT,

  created_at TIMESTAMPTZ DEFAULT NOW(),
  responded_at TIMESTAMPTZ
);

CREATE INDEX idx_inquiries_customer_id ON inquiries(customer_id);
CREATE INDEX idx_inquiries_operator_id ON inquiries(operator_id);
CREATE INDEX idx_inquiries_status ON inquiries(status);
CREATE INDEX idx_inquiries_created_at ON inquiries(created_at DESC);
```

---

### 10. viewings（内見予約） ⭐ NEW

```sql
CREATE TABLE viewings (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  customer_id UUID REFERENCES customers(id) ON DELETE SET NULL,
  property_id UUID NOT NULL REFERENCES properties(id) ON DELETE CASCADE,
  operator_id UUID NOT NULL REFERENCES operators(id) ON DELETE CASCADE, -- ⭐ NEW

  -- 予約日時
  viewing_date TIMESTAMPTZ NOT NULL,
  duration_minutes INTEGER DEFAULT 30, -- 所要時間（分）

  -- ステータス
  status VARCHAR(20) DEFAULT 'confirmed' CHECK (status IN ('confirmed', 'completed', 'cancelled', 'no_show')),

  -- 担当者
  assigned_to UUID REFERENCES auth.users(id), -- オペレーター or 管理者

  -- メモ
  notes TEXT,
  customer_notes TEXT, -- 顧客からの特記事項

  -- メタ情報
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW(),
  cancelled_at TIMESTAMPTZ,
  cancellation_reason TEXT
);

CREATE INDEX idx_viewings_customer_id ON viewings(customer_id);
CREATE INDEX idx_viewings_property_id ON viewings(property_id);
CREATE INDEX idx_viewings_operator_id ON viewings(operator_id);
CREATE INDEX idx_viewings_viewing_date ON viewings(viewing_date);
CREATE INDEX idx_viewings_status ON viewings(status);
CREATE INDEX idx_viewings_assigned_to ON viewings(assigned_to);

-- 更新日時の自動更新
CREATE TRIGGER update_viewings_updated_at
BEFORE UPDATE ON viewings
FOR EACH ROW
EXECUTE FUNCTION update_updated_at_column();
```

---

### 11. email_logs（メール送信ログ） ⭐ NEW

```sql
CREATE TABLE email_logs (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),

  -- 送信先情報
  to_email VARCHAR(255) NOT NULL,
  to_name VARCHAR(100),

  -- メール内容
  from_email VARCHAR(255) NOT NULL DEFAULT 'noreply@ieguchi.jp',
  from_name VARCHAR(100) DEFAULT 'イエグチ',
  subject VARCHAR(255) NOT NULL,
  template_name VARCHAR(100), -- 使用したテンプレート名

  -- 関連情報
  operator_id UUID REFERENCES operators(id) ON DELETE CASCADE, -- ⭐ NEW
  customer_id UUID REFERENCES customers(id) ON DELETE SET NULL,
  property_id UUID REFERENCES properties(id) ON DELETE SET NULL,
  viewing_id UUID REFERENCES viewings(id) ON DELETE SET NULL,
  inquiry_id UUID REFERENCES inquiries(id) ON DELETE SET NULL,

  -- メール種別
  email_type VARCHAR(50) NOT NULL CHECK (email_type IN (
    'viewing_confirmation',     -- 内見予約確定
    'viewing_reminder',         -- 内見リマインダー
    'viewing_cancellation',     -- 内見キャンセル
    'inquiry_confirmation',     -- 問い合わせ確認
    'inquiry_response',         -- 問い合わせ返信
    'system_announcement',      -- システムお知らせ
    'new_property_alert',       -- 新着物件通知
    'password_reset',           -- パスワードリセット
    'email_verification',       -- メール確認
    'other'                     -- その他
  )),

  -- 送信状態
  status VARCHAR(20) DEFAULT 'pending' CHECK (status IN ('pending', 'sent', 'failed', 'bounced')),
  resend_id VARCHAR(100), -- Resend APIのメッセージID
  error_message TEXT,

  -- メタ情報
  sent_at TIMESTAMPTZ,
  created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_email_logs_operator_id ON email_logs(operator_id);
CREATE INDEX idx_email_logs_to_email ON email_logs(to_email);
CREATE INDEX idx_email_logs_customer_id ON email_logs(customer_id);
CREATE INDEX idx_email_logs_email_type ON email_logs(email_type);
CREATE INDEX idx_email_logs_status ON email_logs(status);
CREATE INDEX idx_email_logs_sent_at ON email_logs(sent_at DESC);
CREATE INDEX idx_email_logs_created_at ON email_logs(created_at DESC);
```

---

## Row Level Security（RLS）設定

### operators テーブル ⭐ NEW

```sql
-- RLS有効化
ALTER TABLE operators ENABLE ROW LEVEL SECURITY;

-- オペレーター：自分の企業情報のみ閲覧・更新
CREATE POLICY "operator_own_data"
ON operators FOR ALL
TO authenticated
USING (
  user_id = auth.uid()
  AND (SELECT role FROM auth.users WHERE id = auth.uid()) = 'operator'
);

-- 管理者：すべてのオペレーター情報を閲覧・管理
CREATE POLICY "admin_all_operators"
ON operators FOR ALL
TO authenticated
USING (
  (SELECT role FROM auth.users WHERE id = auth.uid()) = 'admin'
);
```

---

### properties テーブル

```sql
-- RLS有効化
ALTER TABLE properties ENABLE ROW LEVEL SECURITY;

-- 顧客：自分が所属するオペレーターの公開物件のみ閲覧
CREATE POLICY "customer_view_own_operator_properties"
ON properties FOR SELECT
TO authenticated
USING (
  is_public = true
  AND operator_id IN (
    SELECT operator_id FROM customers WHERE user_id = auth.uid()
  )
  AND (SELECT role FROM auth.users WHERE id = auth.uid()) = 'customer'
);

-- オペレーター：自社の物件を閲覧・更新
CREATE POLICY "operator_manage_own_properties"
ON properties FOR ALL
TO authenticated
USING (
  operator_id IN (
    SELECT id FROM operators WHERE user_id = auth.uid()
  )
  AND (SELECT role FROM auth.users WHERE id = auth.uid()) = 'operator'
)
WITH CHECK (
  operator_id IN (
    SELECT id FROM operators WHERE user_id = auth.uid()
  )
  AND (SELECT role FROM auth.users WHERE id = auth.uid()) = 'operator'
);

-- 管理者：すべての物件を閲覧・管理
CREATE POLICY "admin_all_properties"
ON properties FOR ALL
TO authenticated
USING (
  (SELECT role FROM auth.users WHERE id = auth.uid()) = 'admin'
);
```

### property_import_logs テーブル

```sql
ALTER TABLE property_import_logs ENABLE ROW LEVEL SECURITY;

-- オペレーター：自社の取り込みログのみ閲覧
CREATE POLICY "operator_view_own_logs"
ON property_import_logs FOR SELECT
TO authenticated
USING (
  operator_id IN (
    SELECT id FROM operators WHERE user_id = auth.uid()
  )
  AND (SELECT role FROM auth.users WHERE id = auth.uid()) = 'operator'
);

-- 管理者：すべてのログを閲覧
CREATE POLICY "admin_view_all_logs"
ON property_import_logs FOR SELECT
TO authenticated
USING (
  (SELECT role FROM auth.users WHERE id = auth.uid()) = 'admin'
);
```

### customers テーブル

```sql
ALTER TABLE customers ENABLE ROW LEVEL SECURITY;

-- 顧客：自分の情報のみ閲覧・更新
CREATE POLICY "customer_own_data"
ON customers FOR ALL
TO authenticated
USING (
  user_id = auth.uid()
  AND (SELECT role FROM auth.users WHERE id = auth.uid()) = 'customer'
);

-- オペレーター：自社の顧客情報のみ閲覧・管理
CREATE POLICY "operator_view_own_customers"
ON customers FOR ALL
TO authenticated
USING (
  operator_id IN (
    SELECT id FROM operators WHERE user_id = auth.uid()
  )
  AND (SELECT role FROM auth.users WHERE id = auth.uid()) = 'operator'
);

-- 管理者：すべて閲覧・管理可能
CREATE POLICY "admin_all_customers"
ON customers FOR ALL
TO authenticated
USING (
  (SELECT role FROM auth.users WHERE id = auth.uid()) = 'admin'
);
```

### conversations & messages

```sql
ALTER TABLE conversations ENABLE ROW LEVEL SECURITY;
ALTER TABLE messages ENABLE ROW LEVEL SECURITY;

-- 顧客：自分の会話のみ閲覧
CREATE POLICY "customer_own_conversations"
ON conversations FOR SELECT
TO authenticated
USING (
  customer_id IN (
    SELECT id FROM customers WHERE user_id = auth.uid()
  )
  AND (SELECT role FROM auth.users WHERE id = auth.uid()) = 'customer'
);

-- オペレーター：自社の顧客の会話のみ閲覧
CREATE POLICY "operator_view_own_conversations"
ON conversations FOR SELECT
TO authenticated
USING (
  operator_id IN (
    SELECT id FROM operators WHERE user_id = auth.uid()
  )
  AND (SELECT role FROM auth.users WHERE id = auth.uid()) = 'operator'
);

-- 顧客：自分の会話メッセージのみ閲覧
CREATE POLICY "customer_own_messages"
ON messages FOR SELECT
TO authenticated
USING (
  conversation_id IN (
    SELECT c.id FROM conversations c
    JOIN customers cu ON c.customer_id = cu.id
    WHERE cu.user_id = auth.uid()
  )
  AND (SELECT role FROM auth.users WHERE id = auth.uid()) = 'customer'
);

-- オペレーター：自社の顧客の会話メッセージのみ閲覧
CREATE POLICY "operator_view_own_messages"
ON messages FOR SELECT
TO authenticated
USING (
  conversation_id IN (
    SELECT c.id FROM conversations c
    WHERE c.operator_id IN (
      SELECT id FROM operators WHERE user_id = auth.uid()
    )
  )
  AND (SELECT role FROM auth.users WHERE id = auth.uid()) = 'operator'
);

-- 管理者：すべて閲覧可能
CREATE POLICY "admin_view_all_conversations"
ON conversations FOR SELECT
TO authenticated
USING (
  (SELECT role FROM auth.users WHERE id = auth.uid()) = 'admin'
);

CREATE POLICY "admin_view_all_messages"
ON messages FOR SELECT
TO authenticated
USING (
  (SELECT role FROM auth.users WHERE id = auth.uid()) = 'admin'
);
```

---

## pgvector による類似物件検索

### ベクトル検索関数

```sql
CREATE OR REPLACE FUNCTION match_properties(
  query_embedding vector(1536),
  match_threshold float DEFAULT 0.8,
  match_count int DEFAULT 10
)
RETURNS TABLE (
  id uuid,
  title varchar,
  address text,
  rent integer,
  layout varchar,
  similarity float
)
LANGUAGE plpgsql
AS $$
BEGIN
  RETURN QUERY
  SELECT
    p.id,
    p.title,
    p.address,
    p.rent,
    p.layout,
    1 - (p.embedding <=> query_embedding) AS similarity
  FROM properties p
  WHERE 1 - (p.embedding <=> query_embedding) > match_threshold
    AND p.is_public = true
  ORDER BY p.embedding <=> query_embedding
  LIMIT match_count;
END;
$$;
```

### 使用例

```typescript
// 顧客の希望条件をベクトル化
const queryText = "文京区で2LDK、家賃15万円以内のペット可物件";
const embedding = await openai.embeddings.create({
  model: 'text-embedding-3-small',
  input: queryText,
});

// Supabaseで類似検索
const { data } = await supabase.rpc('match_properties', {
  query_embedding: embedding.data[0].embedding,
  match_threshold: 0.8,
  match_count: 10,
});
```

---

## 全文検索

### 全文検索関数

```sql
CREATE OR REPLACE FUNCTION search_properties_fulltext(
  search_query text
)
RETURNS TABLE (
  id uuid,
  title varchar,
  address text,
  rent integer,
  layout varchar,
  rank float
)
LANGUAGE plpgsql
AS $$
BEGIN
  RETURN QUERY
  SELECT
    p.id,
    p.title,
    p.address,
    p.rent,
    p.layout,
    ts_rank(
      to_tsvector('japanese', p.title || ' ' || p.description),
      plainto_tsquery('japanese', search_query)
    ) AS rank
  FROM properties p
  WHERE to_tsvector('japanese', p.title || ' ' || p.description) @@
        plainto_tsquery('japanese', search_query)
    AND p.is_public = true
  ORDER BY rank DESC;
END;
$$;
```

---

## トリガー・関数

### 更新日時の自動更新

```sql
CREATE OR REPLACE FUNCTION update_updated_at_column()
RETURNS TRIGGER AS $$
BEGIN
  NEW.updated_at = NOW();
  RETURN NEW;
END;
$$ LANGUAGE plpgsql;

-- properties テーブルに適用
CREATE TRIGGER update_properties_updated_at
BEFORE UPDATE ON properties
FOR EACH ROW
EXECUTE FUNCTION update_updated_at_column();
```

### 会話終了時のメッセージ数カウント

```sql
CREATE OR REPLACE FUNCTION update_conversation_message_count()
RETURNS TRIGGER AS $$
BEGIN
  UPDATE conversations
  SET total_messages = (
    SELECT COUNT(*) FROM messages WHERE conversation_id = NEW.conversation_id
  )
  WHERE id = NEW.conversation_id;
  RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER count_messages_on_insert
AFTER INSERT ON messages
FOR EACH ROW
EXECUTE FUNCTION update_conversation_message_count();
```

---

## データ保持ポリシー

### 古いデータの削除

```sql
-- 1年以上前の会話ログを削除
CREATE OR REPLACE FUNCTION cleanup_old_conversations()
RETURNS void AS $$
BEGIN
  DELETE FROM conversations
  WHERE started_at < NOW() - INTERVAL '1 year';
END;
$$ LANGUAGE plpgsql;

-- 定期実行（Supabase Cron Jobs）
SELECT cron.schedule(
  'cleanup-old-conversations',
  '0 3 * * 0', -- 毎週日曜 3:00AM
  'SELECT cleanup_old_conversations();'
);
```

---

## バックアップ戦略

### Supabase自動バックアップ
- 毎日自動バックアップ（過去7日分保持）
- Point-in-Time Recovery (PITR) 対応

### 手動バックアップ

```bash
# pg_dumpでエクスポート
pg_dump -h <supabase-host> -U postgres -d postgres > backup.sql

# 復元
psql -h <supabase-host> -U postgres -d postgres < backup.sql
```

---

次のドキュメント: [API設計](./api-design.md)
