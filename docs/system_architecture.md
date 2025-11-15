# IEGUCHI システム全体像・アーキテクチャ設計書

> 機能仕様に関する認識合わせ用ドキュメント

---

## 目次

1. [システム概要](#システム概要)
2. [システム全体構成](#システム全体構成)
3. [主要機能フロー](#主要機能フロー)
4. [画面構成](#画面構成)
5. [技術スタック](#技術スタック)
6. [セキュリティ対策](#セキュリティ対策)
7. [システムの特徴](#システムの特徴)

---

## システム概要

IEGUCHIは、不動産業界向けの窓口営業支援AIアシスタントシステムです。AIアバター「イエグチさん」が顧客対応を行い、24時間365日の物件検索・提案を実現します。

### 主な機能

- AI会話による物件検索・提案
- 音声入力対応（高齢者にも配慮）
- アバターによる視覚的なコミュニケーション
- Chrome拡張機能による効率的な物件登録
- 内見予約の自動化とメール通知

---

## システム全体構成

### ユーザー種別と権限

**階層構造**:
```
Admin（gyam様）
  └─ Operator（不動産会社A、B、C...）
       └─ Customer（入居者）
```

```mermaid
graph TB
    subgraph Users["ユーザー種別"]
        Admin["管理者（gyam様）<br/>・オペレーター企業管理<br/>・全データ閲覧<br/>・サブスク管理<br/>・システム設定"]
        Operator["オペレーター（不動産会社）<br/>・物件取り込み<br/>・自社物件編集<br/>・自社顧客管理<br/>・対応履歴確認"]
        Customer["顧客（入居者）<br/>・物件検索<br/>・AI対話<br/>・音声入力<br/>・内見予約"]
    end

    subgraph Interfaces["アクセス方法"]
        Web["Webブラウザ"]
        Extension["Chrome拡張"]
        OperatorDash["オペレーター<br/>管理画面"]
        AdminDash["管理者<br/>管理画面"]
    end

    Admin --> AdminDash
    Admin -.管理.-> Operator
    Operator --> Extension
    Operator --> OperatorDash
    Operator -.管理.-> Customer
    Customer --> Web
```

### システムアーキテクチャ

```mermaid
graph TB
    subgraph Frontend["フロントエンド層"]
        LP["ランディングページ"]
        CustomerUI["顧客対応画面"]
        PropertyDetail["物件詳細ページ"]
        AdminUI["管理画面"]
        ChromeExt["Chrome拡張機能"]
    end
    
    subgraph AILayer["AI処理層"]
        Avatar["AIアバターシステム<br/>・表情切り替え（5種類）<br/>・Framer Motionアニメーション"]
        Voice["音声処理システム<br/>・Whisper API（音声認識）<br/>・ElevenLabs API（音声合成）"]
        ChatEngine["AI会話エンジン（GPT-4o）<br/>・条件ヒアリング<br/>・物件検索<br/>・感情分析<br/>・RAG検索"]
        PropertyMgmt["物件データ管理<br/>・GPT-4oによる構造化抽出<br/>・HTMLスナップショット保存"]
    end
    
    subgraph NotificationLayer["通知層"]
        Email["メール通知システム<br/>・内見予約確定メール<br/>・カレンダー招待（.ics）<br/>・問い合わせ確認メール<br/>・Resend API + React Email"]
    end
    
    subgraph DataLayer["データ層"]
        DB[(Supabase PostgreSQL<br/>・物件データ<br/>・顧客データ<br/>・会話履歴<br/>・内見予約<br/>・メール送信ログ)]
    end
    
    subgraph External["外部サービス"]
        OpenAI["OpenAI API"]
        Whisper["Whisper API"]
        ElevenLabs["ElevenLabs API"]
        Resend["Resend"]
        Vercel["Vercel"]
        Sentry["Sentry"]
    end
    
    Frontend --> AILayer
    AILayer --> NotificationLayer
    AILayer --> DataLayer
    NotificationLayer --> DataLayer
    
    ChatEngine --> OpenAI
    Voice --> Whisper
    Voice --> ElevenLabs
    Email --> Resend
    Frontend --> Vercel
    Frontend --> Sentry
```

---

## 主要機能フロー

### 1. 顧客の物件検索フロー

```mermaid
sequenceDiagram
    actor Customer as 顧客
    participant LP as ランディングページ
    participant UI as 顧客対応画面
    participant Whisper as Whisper API
    participant GPT as GPT-4o
    participant ElevenLabs as ElevenLabs API
    participant DB as データベース
    
    Customer->>LP: Webサイトアクセス
    LP->>UI: 「物件を探す」クリック
    Customer->>UI: 音声 or テキスト入力<br/>「文京区で2LDK、家賃15万円以内」
    
    alt 音声入力の場合
        UI->>Whisper: 音声データ送信
        Whisper->>UI: テキスト化
        UI->>Customer: テキスト確認・修正可能
    end
    
    UI->>GPT: 条件送信
    GPT->>DB: 物件検索
    DB->>GPT: 物件データ
    GPT->>UI: 提案文生成<br/>感情分析結果
    
    UI->>ElevenLabs: テキスト送信
    ElevenLabs->>UI: 音声データ
    
    UI->>Customer: レスポンス表示<br/>・アバター表情切り替え<br/>・音声再生<br/>・物件カード表示
    
    Customer->>UI: 物件カードクリック
    UI->>Customer: 物件詳細ページ表示
```

### 2. オペレーターの物件取り込みフロー

```mermaid
sequenceDiagram
    actor Operator as オペレーター
    participant Site as 物件サイト<br/>（アットホーム等）
    participant Ext as Chrome拡張機能
    participant API as IEGUCHIシステム
    participant GPT as GPT-4o
    participant DB as データベース
    
    Operator->>Site: 物件ページ表示
    Operator->>Ext: 拡張機能アイコンクリック
    Ext->>Operator: ポップアップ表示<br/>「この物件を取り込みますか？」
    Operator->>Ext: 取り込みボタンクリック
    
    Ext->>API: HTMLデータ + Auth Token送信
    API->>API: 権限確認
    API->>GPT: HTML送信
    GPT->>API: 構造化データ抽出<br/>{title, address, rent, layout...}
    
    API->>DB: データ保存<br/>・propertiesテーブル<br/>・HTMLスナップショット<br/>・取り込みログ
    
    DB->>API: 保存完了
    API->>Ext: 完了通知
    Ext->>Operator: 「取り込み完了しました」
```

### 3. 内見予約・メール通知フロー

```mermaid
sequenceDiagram
    actor Customer as 顧客
    participant UI as 物件詳細ページ
    participant API as IEGUCHIシステム
    participant DB as データベース
    participant Resend as Resend API
    participant Email as メールクライアント
    actor Operator as オペレーター
    
    Customer->>UI: 「内見予約」クリック
    UI->>Customer: 内見予約フォーム表示
    Customer->>UI: 情報入力<br/>（希望日時、連絡先、備考）
    Customer->>UI: 「予約する」クリック
    
    UI->>API: 予約データ送信
    API->>DB: viewingsテーブルに保存
    
    API->>Resend: メール送信リクエスト<br/>・顧客宛て<br/>・オペレーター宛て
    Resend->>Resend: カレンダー招待生成（.ics）<br/>HTMLメール作成
    
    Resend->>Email: 顧客に送信
    Resend->>Email: オペレーターに送信
    
    Resend->>API: 送信完了
    API->>DB: email_logsに記録
    
    Email->>Customer: 内見予約確定メール受信<br/>+ calendar.ics添付
    Email->>Operator: 内見予約通知受信<br/>+ calendar.ics添付
```

### 4. 管理者のダッシュボード利用フロー

```mermaid
graph TD
    Admin[管理者<br/>gyam様] --> Login[管理画面ログイン]
    Login --> Dashboard[ダッシュボード]

    Dashboard --> Stats[統計情報<br/>・システム全体の会話数<br/>・オペレーター企業数<br/>・物件総数<br/>・顧客総数]
    Dashboard --> Activity[最近の活動<br/>・新規オペレーター登録<br/>・物件取り込みログ<br/>・顧客問い合わせ]

    Dashboard --> OperatorMgmt[オペレーター管理<br/>・不動産会社の登録<br/>・サブスク状態管理<br/>・契約管理]
    Dashboard --> PropertyMgmt[全物件管理<br/>・全オペレーターの物件閲覧<br/>・編集・削除]
    Dashboard --> CustomerMgmt[全顧客管理<br/>・全オペレーターの顧客閲覧<br/>・監査用]
    Dashboard --> Logs[ログ確認<br/>・監査ログ<br/>・エラーログ<br/>・アクセスログ]
    Dashboard --> Settings[設定<br/>・システム設定<br/>・API Key管理]
```

---

## 画面構成

本システムは以下の7つの主要画面で構成されます。

### 1. ランディングページ（LP）

**目的**: サービスの特徴を紹介し、顧客を物件検索へ誘導

**主要要素**:
- サービスロゴとキャッチコピー
- 主要機能の紹介（24時間対応、音声検索、高齢者対応）
- 「物件を探す」CTA（Call To Action）ボタン

### 2. 顧客対応画面（メイン画面）

**目的**: AIアバターとの対話による物件検索

**主要要素**:
- AIアバター「イエグチさん」の表示
- 会話履歴エリア
- 音声入力ボタン
- テキスト入力フォーム
- 物件カード表示エリア（検索結果）

### 3. 物件詳細ページ

**目的**: 選択した物件の詳細情報を表示

**主要要素**:
- 物件写真スライダー
- 基本情報（家賃、間取り、住所）
- 詳細情報（設備、アクセス、間取り図）
- 内見予約ボタン
- 問い合わせボタン

### 4. 物件一覧ページ

**目的**: 条件に合致する物件の一覧表示と絞り込み

**主要要素**:
- 検索条件フィルター（エリア、家賃、間取り等）
- 検索結果件数表示
- 物件カード一覧
- ページネーション

### 5. 管理画面（オペレーター用）

**目的**: オペレーターによる自社の物件・顧客管理

**主要要素**:
- 自社が取り込んだ物件一覧
- 物件の編集・削除機能
- 取り込みログ表示
- 自社の顧客一覧・管理
- 自社の顧客対応履歴
- 自社の内見予約管理
- 自社の問い合わせ管理

**制限事項**:
- 他社の物件・顧客データは閲覧不可
- システム設定の変更不可

### 6. ダッシュボード（管理者用 - gyam様専用）

**目的**: システム全体の状況把握とオペレーター企業管理

**主要要素**:
- システム全体の統計情報
  - オペレーター企業数（active, suspended, cancelled）
  - 全物件数・全顧客数
  - 会話数推移
- オペレーター管理機能
  - 不動産会社の登録・編集・削除
  - サブスクリプション状態管理
  - 契約管理
- 全データ閲覧
  - 全オペレーターの物件データ
  - 全オペレーターの顧客データ（監査用）
- グラフ表示（会話数推移、物件取り込み推移等）
- 最近の活動ログ
- 各管理機能へのナビゲーション

### 7. 設定ページ

**目的**: システムの各種設定とカスタマイズ

**主要要素**:
- アバター設定（表情切り替え速度、デフォルト表情）
- 音声設定（速度、音量）
- API設定（管理者のみ: OpenAI API Key、ElevenLabs API Key等）

---

## 技術スタック

### フロントエンド

| 技術 | バージョン | 用途 |
|------|-----------|------|
| Next.js | 15 | Reactフレームワーク |
| React | 19 | UIライブラリ |
| TypeScript | 5.x | 型安全な開発 |
| Tailwind CSS | 3.x | スタイリング |
| shadcn/ui | - | UIコンポーネント |
| Framer Motion | - | アニメーション |

### バックエンド

| 技術 | 用途 |
|------|------|
| Supabase | データベース（PostgreSQL）、認証、ストレージ |
| Vercel | ホスティング、サーバーレス関数 |

### AI・音声処理

| サービス | 用途 |
|---------|------|
| OpenAI GPT-4o | AI会話エンジン、構造化データ抽出 |
| Whisper API | 音声認識（音声→テキスト） |
| ElevenLabs API | 音声合成（テキスト→音声） |

### 通知・メール

| サービス | 用途 |
|---------|------|
| Resend | メール配信 |
| React Email | HTMLメールテンプレート |

### 監視・ログ

| サービス | 用途 |
|---------|------|
| Sentry | エラー監視 |
| Axiom | ログ管理 |

### Chrome拡張機能

| 技術 | 用途 |
|------|------|
| Manifest V3 | Chrome拡張機能の仕様 |
| TypeScript | 拡張機能のロジック |

---

## セキュリティ対策

### 認証・認可

**実装内容**:
- Supabase Authによるユーザー認証（JWT）
- Row Level Security（RLS）による細密な権限制御
- マルチテナント構造によるデータ分離
- ロールベースアクセス制御（RBAC）
  - **admin（gyam様）**: システム全体の管理者
    - オペレーター企業の登録・管理
    - 全オペレーターのデータ閲覧
    - サブスクリプション管理
    - システム設定管理
  - **operator（不動産会社）**: 契約不動産会社
    - 自社物件の取り込み・編集
    - 自社顧客の管理
    - 自社データのみアクセス可能
    - 他社データは完全に分離
  - **customer（入居者）**: エンドユーザー
    - 物件検索、問い合わせ、内見予約
    - 自分が所属するオペレーターの物件のみ閲覧

### データ保護

**実装内容**:
- HTTPS通信の強制
- データベースの暗号化（at rest）
- 通信データの暗号化（in transit）
- 機密情報（API Key等）の環境変数管理

### 監査・ログ

**実装内容**:
- 物件取り込みログの記録（法的証跡）
- HTMLスナップショットの保存（取り込み時点の証拠）
- ユーザーアクセスログの記録
- セキュリティイベントの監視

### コンプライアンス

**対応項目**:
- 個人情報保護法への対応
- 不動産広告規制への準拠
- データ保持期間の設定と自動削除

---

## システムの特徴

### 実現可能な機能

- ✅ 24時間365日のAI自動物件提案
- ✅ 音声入力による物件検索（高齢者対応）
- ✅ 感情表現豊かなアバター表示（5種類の表情）
- ✅ Chrome拡張機能によるワンクリック物件取り込み
- ✅ マルチテナント構造による複数不動産会社の管理
- ✅ オペレーター企業のサブスクリプション管理
- ✅ 完全なデータ分離（他社データへのアクセス不可）
- ✅ ロールベースのセキュアな権限管理
- ✅ 会話履歴の記録と分析
- ✅ 内見予約の自動メール通知
- ✅ カレンダー招待ファイル（.ics）の自動生成
- ✅ 問い合わせ受付確認メールの自動送信
- ✅ システムお知らせメールの一斉配信

### 実現できない機能

- ❌ 3Dアバター（2D静止画のみ対応）
- ❌ リアルタイム動画通話機能
- ❌ 完全自動の物件取り込み（オペレーターの操作が必要）
- ❌ オペレーターのスケジュール自動調整（予約受付・通知まで）

---

## 次のステップ

### ご確認いただきたい事項

本ドキュメントは、システムの機能仕様に関する認識の相違がないかを確認するためのものです。以下の点についてご確認をお願いいたします。

1. **システム仕様の確認**
   - 記載された機能要件に過不足がないか
   - 追加・変更のご要望がないか
   - 実現できない機能についての認識が一致しているか

2. **技術スタックの確認**
   - 採用する技術に問題がないか
   - 長期運用の観点からの懸念点がないか

3. **画面構成・UIの確認**
   - 必要な画面がすべて含まれているか
   - 各画面の機能要件が適切か

ご質問やご不明な点、変更のご要望等がございましたら、お気軽にご連絡ください。

---

**発行者**: 小松原遼羽  
**メールアドレス**: r.kono17y204@gmail.com  
**電話番号**: 070-1317-5266  
**発行日**: 2025年11月15日  
**バージョン**: 1.0
