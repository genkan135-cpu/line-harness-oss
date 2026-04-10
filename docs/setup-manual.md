# LINE Harness OSS セットアップマニュアル

> **対象**: LINE Harness OSS を初めてデプロイする開発者・AIエージェント  
> **所要時間**: 約60〜90分（LINE Developers Consoleの設定含む）  
> **前提**: macOS または Linux、Node.js 18以上、pnpm インストール済み

---

## 目次

1. [アーキテクチャ概要](#1-アーキテクチャ概要)
2. [前提条件の確認](#2-前提条件の確認)
3. [LINE Developers Console 設定](#3-line-developers-console-設定)
4. [リポジトリ取得と依存インストール](#4-リポジトリ取得と依存インストール)
5. [Cloudflare 初期設定](#5-cloudflare-初期設定)
6. [D1 データベース作成とスキーマ適用](#6-d1-データベース作成とスキーマ適用)
7. [シークレット登録](#7-シークレット登録)
8. [ビルドとデプロイ](#8-ビルドとデプロイ)
9. [管理ユーザー作成](#9-管理ユーザー作成)
10. [Webhook 設定](#10-webhook-設定)
11. [LIFF セットアップ](#11-liff-セットアップ)
12. [管理画面（Admin UI）起動](#12-管理画面admin-ui起動)
13. [動作確認](#13-動作確認)
14. [API リファレンス](#14-api-リファレンス)
15. [フォームと条件付きタグ](#15-フォームと条件付きタグ)
16. [マルチアカウント運用](#16-マルチアカウント運用)
17. [トラブルシューティング一覧](#17-トラブルシューティング一覧)

---

## 1. アーキテクチャ概要

```
LINE Platform ──→ Cloudflare Workers (Hono) ──→ D1 (SQLite)
                         │                         │
                         ├── R2 (画像ストレージ)    │
                         ├── LIFF (フォーム)        │
                         │                         │
                  Admin UI (Next.js) ←─── API ────┘
                  apps/web (port 3001)
```

**リポジトリ構成（pnpm モノレポ）**:

| ディレクトリ | 役割 |
|---|---|
| `apps/worker` | Cloudflare Workers 本体（Hono フレームワーク） |
| `apps/web` | 管理画面 Next.js アプリ（localhost:3001 または Cloudflare Pages） |
| `packages/db` | スキーマ定義（schema.sql + migrations/） |
| `packages/*` | 共有パッケージ |

**認証方式**: APIキーベース（`staff_members` テーブル）。パスワード認証ではない。

---

## 2. 前提条件の確認

以下がインストール済みであることを確認する。

```bash
node -v   # v18 以上
pnpm -v   # pnpm がインストールされていること
```

> **PITFALL: pnpm 必須**  
> このリポジトリは pnpm ワークスペースを使用している。`npm install` では依存関係が正しく解決されない。  
> 未インストールの場合: `brew install pnpm`（macOS）または `npm install -g pnpm`

---

## 3. LINE Developers Console 設定

LINE Developers Console（https://developers.line.biz/console/）で以下を作成する。

### 3-1. プロバイダー作成

1. LINE Developers Console にログイン
2. 「プロバイダー」を新規作成（例: 「あなたの組織名」）
3. **このプロバイダー名を記録しておく**（後の LIFF 設定で重要）

### 3-2. Messaging API チャネル作成

1. 作成したプロバイダーの中で「Messaging API」チャネルを新規作成
2. 以下を取得して控える:
   - **Channel Secret**（チャネル基本設定タブ）
   - **Channel Access Token（長期）**（Messaging API タブ → 「発行」ボタン）

> **PITFALL: 既存LINE公式アカウントとの競合**  
> 既にAutoSNS等のサービスが Webhook URL を設定しているLINE公式アカウントを流用すると、Webhook URLを切り替えた時点でAutoSNSが無効になる。LINEは1アカウントにつき1つのWebhook URLしか許可しない。LINE Harness 用に新しいLINE公式アカウントを作成することを推奨する。

---

## 4. リポジトリ取得と依存インストール

```bash
git clone https://github.com/shudesu/line-harness-oss.git
cd line-harness-oss
pnpm install
```

---

## 5. Cloudflare 初期設定

### 5-1. wrangler ログイン

```bash
npx wrangler login
```

ブラウザが自動的に開き、Cloudflare のOAuth認証画面が表示される。

> **PITFALL: ブラウザが自動で開かない場合**  
> Claude Code やSSH環境など、一部のターミナル環境ではブラウザが自動で開かない。ターミナルに表示されるOAuth URLをコピーし、手動でブラウザに貼り付けて認証を完了すること。

> **PITFALL: OAuth トークン期限切れ**  
> wrangler の OAuth トークンは期限がある。D1コマンド等が `Authentication error [code: 10000]` で失敗した場合は、`npx wrangler login` を再実行する。

### 5-2. workers.dev サブドメインの有効化

初めて Cloudflare Workers を使う場合、workers.dev サブドメインの登録が必要。

> **PITFALL: "You need to register a workers.dev subdomain" エラー**  
> 初回デプロイ時にこのエラーが出た場合:
> 1. Cloudflare ダッシュボード → Workers & Pages に移動
> 2. workers.dev サブドメインを有効化（オンボーディング画面が表示される場合はそれに従う）
> 3. 有効化後に再度デプロイを試行

---

## 6. D1 データベース作成とスキーマ適用

### 6-1. D1 データベース作成

```bash
npx wrangler d1 create line-harness
```

出力に表示される `database_id` を控える。

### 6-2. wrangler.toml 編集

`apps/worker/wrangler.toml` を編集:

```toml
account_id = "ここにCloudflareアカウントIDを設定"

[[d1_databases]]
binding = "DB"
database_name = "line-harness"
database_id = "ここにD1のdatabase_idを設定"
```

> **PITFALL: R2 バケット未有効化**  
> `wrangler.toml` に `[[r2_buckets]]` セクションがある場合、Cloudflare ダッシュボードで R2 を有効化していないとデプロイが失敗する。初回セットアップ時は `[[r2_buckets]]` セクションをコメントアウトすることを推奨する:
> ```toml
> # [[r2_buckets]]
> # binding = "IMAGES"
> # bucket_name = "line-harness-images"
> ```

> **PITFALL: マルチ環境の警告**  
> `wrangler.toml` にはデフォルト環境と production 環境の2つが定義されている。コマンド実行時に環境に関する警告が出ることがあるが、初回セットアップではデフォルト（トップレベル）環境を使えばよい。

### 6-3. スキーマとマイグレーション適用

**重要: schema.sql だけでは不十分。マイグレーションも全て実行する必要がある。**

```bash
cd apps/worker

# 1. ベーススキーマ適用
npx wrangler d1 execute line-harness --remote --file=../../packages/db/schema.sql

# 2. 全マイグレーション適用（必須）
for f in ../../packages/db/migrations/*.sql; do
  echo "Applying: $f"
  npx wrangler d1 execute line-harness --remote --file="$f"
done
```

> **PITFALL: schema.sql だけでは不完全**  
> schema.sql にはベースのテーブルのみ定義されている。`forms`, `staff_members`（api_key列含む）, `tracked_links` 等の重要なテーブル・列はマイグレーションファイルで追加される。マイグレーションを実行しないと、フォーム機能やAPI認証が動作しない。

> **PITFALL: 重複カラムエラーは無害**  
> 一部のマイグレーションは schema.sql に既に存在するカラムを追加しようとする。`duplicate column name` エラーが表示されるが、これは想定内であり無害。処理を続行してよい。

---

## 7. シークレット登録

以下のシークレットを Cloudflare Workers に登録する。

```bash
cd apps/worker

# LINE Channel Secret
echo "YOUR_CHANNEL_SECRET" | npx wrangler secret put LINE_CHANNEL_SECRET

# LINE Channel Access Token（長期トークン）
echo "YOUR_CHANNEL_ACCESS_TOKEN" | npx wrangler secret put LINE_CHANNEL_ACCESS_TOKEN
```

> **PITFALL: 非インタラクティブ環境での secret 登録**  
> `npx wrangler secret put` はデフォルトでインタラクティブ入力を求める。CI/CDやAIエージェント環境では `echo "VALUE" | npx wrangler secret put SECRET_NAME` のパイプ形式を使うこと。

---

## 8. ビルドとデプロイ

### 8-1. ビルド

```bash
cd apps/worker

# LIFF_ID は後で設定するため、初回は仮値でビルド可能
VITE_LIFF_ID=placeholder pnpm --filter worker build
```

> **PITFALL: VITE_LIFF_ID はビルド時に必要**  
> `VITE_LIFF_ID` は Vite のビルドプロセスでクライアントJSに埋め込まれる。ランタイム環境変数ではなく、ビルド時に設定する必要がある。LIFF設定完了後に正しい値で再ビルド・再デプロイすること。

### 8-2. デプロイ

```bash
cd apps/worker
npx wrangler deploy
```

デプロイ成功後、Worker の URL が表示される（例: `https://line-harness.あなたのサブドメイン.workers.dev`）。この URL を控える。

> **PITFALL: ルート URL（/）は管理画面ではない**  
> Worker の `/` エンドポイントにブラウザでアクセスすると「読み込み中...」のスピナーが表示される。これは LIFF クライアントページであり、LIFF SDK の初期化を試みるため通常のブラウザでは動作しない。管理画面は別の Next.js アプリ（`apps/web`）である。

---

## 9. 管理ユーザー作成

LINE Harness にはセットアップウィザードが存在しない。D1に直接SQLを実行して管理ユーザーを作成する。

```bash
cd apps/worker

npx wrangler d1 execute line-harness --remote --command="INSERT INTO staff_members (id, name, email, role, api_key, is_active) VALUES ('owner-001', 'Admin', 'admin@example.com', 'owner', 'lh_' || hex(randomblob(16)), 1);"

# 生成された API キーを確認
npx wrangler d1 execute line-harness --remote --command="SELECT api_key FROM staff_members WHERE id = 'owner-001';"
```

表示された `lh_xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx` 形式のAPIキーを安全に保管する。これが全てのAPI認証に使用される。

---

## 10. Webhook 設定

LINE Developers Console に戻り、Messaging API チャネルの設定を行う。

1. **Messaging API タブ** → 「Webhook URL」に以下を設定:
   ```
   https://line-harness.あなたのサブドメイン.workers.dev/webhook
   ```
2. 「Webhook の利用」を **オン** にする
3. 「検証」ボタンでWebhookの疎通を確認（成功が表示されればOK）

---

## 11. LIFF セットアップ

LIFF（LINE Front-end Framework）はフォーム機能に必要。フォームを使わない場合はスキップ可能。

### 11-1. LINE Login チャネル作成

> **PITFALL: LIFF は Messaging API チャネルには追加できない**  
> LIFF アプリは LINE Login チャネルにのみ追加可能。Messaging API チャネルとは別に LINE Login チャネルを作成する必要がある。

> **CRITICAL PITFALL: 同一プロバイダー内に作成すること**  
> LINE Login チャネルは必ず **Messaging API チャネルと同じプロバイダー内** に作成すること。異なるプロバイダーに作成すると、LIFF で取得される userId が Messaging API の userId と異なるため、友だちの特定ができなくなる（フォーム送信者が「不明」と表示される）。

1. LINE Developers Console → 先ほどと **同じプロバイダー** を選択
2. 「LINE Login」チャネルを新規作成
3. 以下を控える:
   - **Channel ID**
   - **Channel Secret**

### 11-2. LINE Login チャネルの公開

> **CRITICAL PITFALL: 「開発中」ステータスのままだと動作しない**  
> LINE Login チャネルのデフォルトステータスは「開発中」。この状態では一般ユーザーがアクセスした際に `400 Bad Request: This channel is now developing status. User need to have developer role.` エラーが表示される。

1. LINE Login チャネルの設定画面を開く
2. ステータスを **「開発中」→「公開済み」** に変更

### 11-3. LIFF アプリ作成

LINE Login チャネルのアクセストークンを取得し、LIFF アプリを作成する:

```bash
# LINE Login チャネルのアクセストークン取得
TOKEN=$(curl -s -X POST https://api.line.me/v2/oauth/accessToken \
  -H 'Content-Type: application/x-www-form-urlencoded' \
  -d 'grant_type=client_credentials&client_id=LINE_LOGINのCHANNEL_ID&client_secret=LINE_LOGINのCHANNEL_SECRET' \
  | python3 -c 'import sys,json; print(json.load(sys.stdin)["access_token"])')

echo "Access Token: $TOKEN"

# LIFF アプリ作成
curl -s -X POST https://api.line.me/liff/v1/apps \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "view": {
      "type": "full",
      "url": "https://line-harness.あなたのサブドメイン.workers.dev"
    },
    "description": "Form",
    "scope": ["profile"],
    "botPrompt": "aggressive"
  }'
```

レスポンスに含まれる `liffId` を控える（例: `1234567890-xxxxxxxx`）。

### 11-4. 正しい LIFF_ID で再ビルド・再デプロイ

```bash
cd apps/worker
VITE_LIFF_ID=取得したLIFF_ID pnpm --filter worker build
npx wrangler deploy
```

### 11-5. フォーム URL 形式

フォームの URL は以下の形式になる:

```
https://liff.line.me/{LIFF_ID}?page=form&id={FORM_ID}
```

この URL を LINE メッセージやリッチメニューに設定して使用する。

---

## 12. 管理画面（Admin UI）起動

管理画面は `apps/web` の Next.js アプリとして提供される。

### ローカル起動

```bash
NEXT_PUBLIC_API_URL=https://line-harness.あなたのサブドメイン.workers.dev pnpm --filter web dev
```

ブラウザで `http://localhost:3001` にアクセスし、ステップ9で取得したAPIキーでログインする。

### Cloudflare Pages へのデプロイ（任意）

管理画面を常時公開する場合は Cloudflare Pages にデプロイすることも可能。

---

## 13. 動作確認

### 13-1. ヘルスチェック

```bash
curl -H "Authorization: Bearer lh_あなたのAPIキー" \
  https://line-harness.あなたのサブドメイン.workers.dev/health
```

### 13-2. 友だち追加テスト

1. LINE アプリで作成した LINE 公式アカウントを友だち追加
2. API で友だち一覧を確認:
   ```bash
   curl -H "Authorization: Bearer lh_あなたのAPIキー" \
     https://line-harness.あなたのサブドメイン.workers.dev/api/friends
   ```
3. 友だちが表示されれば Webhook 連携は正常

### 13-3. メッセージ送信テスト

```bash
curl -X POST \
  -H "Authorization: Bearer lh_あなたのAPIキー" \
  -H "Content-Type: application/json" \
  -d '{"content": "LINE Harness セットアップ完了テスト"}' \
  https://line-harness.あなたのサブドメイン.workers.dev/api/friends/{FRIEND_ID}/messages
```

> **PITFALL: Flex Message の送信形式**  
> `messageType: "flex"` でメッセージを送信する場合、`content` には Flex Message のJSON を **文字列化（stringify）** して渡す必要がある。JSON オブジェクトをそのまま渡すと、LINE 上で生の JSON テキストが表示されてしまう。

---

## 14. API リファレンス

全てのAPIエンドポイントは `Authorization: Bearer lh_xxxxx` ヘッダーが必要。

| メソッド | エンドポイント | 説明 |
|---|---|---|
| GET | `/health` | ヘルスチェック |
| GET | `/api/friends` | 友だち一覧取得 |
| POST | `/api/friends/:id/messages` | 個別メッセージ送信 |
| POST | `/api/friends/:id/tags` | 友だちにタグ付与 |
| GET | `/api/tags` | タグ一覧取得 |
| POST | `/api/tags` | タグ作成 |
| GET | `/api/forms` | フォーム一覧取得 |
| POST | `/api/forms` | フォーム作成 |
| GET | `/api/forms/:id/submissions` | フォーム回答一覧 |
| GET | `/api/automations` | オートメーション一覧 |
| POST | `/api/automations` | オートメーション作成 |
| POST | `/api/broadcasts` | ブロードキャスト作成 |
| POST | `/api/broadcasts/:id/send` | ブロードキャスト送信 |
| POST | `/api/staff` | スタッフ追加（owner権限のみ） |

---

## 15. フォームと条件付きタグ

### 15-1. フォーム作成

```bash
curl -X POST \
  -H "Authorization: Bearer lh_あなたのAPIキー" \
  -H "Content-Type: application/json" \
  -d '{
    "title": "アンケート",
    "description": "お客様アンケート",
    "fields": [
      {"name": "interest", "label": "興味のある分野", "type": "select", "options": ["商品A", "商品B", "商品C"], "required": true},
      {"name": "email", "label": "メールアドレス", "type": "email", "required": false}
    ]
  }' \
  https://line-harness.あなたのサブドメイン.workers.dev/api/forms
```

**フォームフィールドの type 一覧**: `text`, `textarea`, `select`, `radio`, `checkbox`, `date`, `email`, `tel`, `number`

### 15-2. 固定タグ付与（on_submit_tag_id）

フォーム送信時に全回答者へ同一タグを付与する場合は、フォーム作成時に `on_submit_tag_id` を指定する。ただし 1 つのタグしか設定できない。

### 15-3. 条件付きタグ付与（オートメーション）

回答内容に応じて異なるタグを付与するにはオートメーションを使用する。

```bash
# 1. タグ作成
curl -X POST \
  -H "Authorization: Bearer lh_あなたのAPIキー" \
  -H "Content-Type: application/json" \
  -d '{"name": "商品A興味あり", "color": "#EF4444"}' \
  https://line-harness.あなたのサブドメイン.workers.dev/api/tags

# 2. オートメーション作成
curl -X POST \
  -H "Authorization: Bearer lh_あなたのAPIキー" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "商品A回答タグ付与",
    "eventType": "form_submitted",
    "conditions": {
      "form_id": "フォームのUUID",
      "field_match": {"interest": "商品A"}
    },
    "actions": [
      {"type": "add_tag", "params": {"tag_id": "タグのUUID"}}
    ]
  }' \
  https://line-harness.あなたのサブドメイン.workers.dev/api/automations
```

> **注意**: OSS版では `form_submitted` イベントのオートメーション実行コードが `apps/worker/src/routes/forms.ts` のフォーム送信後の副作用セクションに追加されている。もし条件付きタグが動作しない場合は、このファイルに `automations` テーブルを参照して `field_match` 条件をマッチングし、`add_tag` アクションを実行するコードが含まれているか確認すること。

---

## 16. マルチアカウント運用

LINE Harness は1つの Worker インスタンスで複数のLINE公式アカウントを管理できる。

- `line_accounts` テーブルに LINE アカウント情報を格納
- Webhook 受信時にチャネル署名で自動ルーティング
- `users` テーブルで複数アカウントをまたいだ同一人物の UUID 管理が可能（`friends.user_id` → `users.id`）
- 管理画面のサイドバーでアカウント切替可能

---

## 17. トラブルシューティング一覧

本文中の PITFALL を含め、発生しやすい問題を一覧にまとめる。

| 症状 | 原因 | 対処 |
|---|---|---|
| `pnpm install` ではなく `npm install` を実行した | pnpm ワークスペース未対応 | `rm -rf node_modules && pnpm install` |
| デプロイ時 "You need to register a workers.dev subdomain" | workers.dev 未有効化 | Cloudflare ダッシュボードで有効化 |
| デプロイ時 R2 関連エラー | R2 未有効化 | wrangler.toml の `[[r2_buckets]]` をコメントアウト |
| D1 コマンドが "Authentication error [code: 10000]" | OAuth トークン期限切れ | `npx wrangler login` を再実行 |
| `wrangler login` でブラウザが開かない | ヘッドレス環境 | ターミナルに表示されるURLを手動でブラウザにコピー |
| API が 401 を返す | staff_members 未作成 or api_key 不正 | ステップ9 の SQL を再確認 |
| フォーム送信者が「不明」 | LIFF が別プロバイダー | LINE Login チャネルを同じプロバイダーに再作成 |
| LIFF で 400 Bad Request | LINE Login チャネルが「開発中」 | ステータスを「公開済み」に変更 |
| `/` にアクセスすると「読み込み中...」スピナー | 正常動作（LIFF ページ） | 管理画面は `apps/web` で別途起動 |
| Flex Message が生JSON で表示される | content の形式不正 | JSON を文字列化して渡す |
| schema.sql 実行後に API が動かない | マイグレーション未適用 | ステップ6-3 の for ループで全 migration を適用 |
| マイグレーションで "duplicate column name" | schema.sql と重複 | 無害。無視してよい |
| `wrangler secret put` が入力待ちで止まる | インタラクティブモード | `echo "VALUE" \| npx wrangler secret put NAME` |
| Webhook 検証が失敗する | URL 未設定 or シークレット不一致 | Webhook URL と LINE_CHANNEL_SECRET を再確認 |
| 既存の AutoSNS が動かなくなった | Webhook URL 競合 | LINE は1アカウント1 Webhook。新規アカウントを使用 |
| 環境に関する wrangler 警告 | マルチ環境定義 | 初回はデフォルト環境（トップレベル）でOK |

---

## 付録: セットアップチェックリスト

セットアップ完了を確認するためのチェックリスト:

- [ ] LINE Developers Console で Messaging API チャネル作成済み
- [ ] Channel Secret と Channel Access Token を取得済み
- [ ] `pnpm install` 完了
- [ ] `npx wrangler login` 完了
- [ ] D1 データベース作成済み
- [ ] `wrangler.toml` に `account_id` と `database_id` を設定済み
- [ ] R2 セクションの対処完了（有効化 or コメントアウト）
- [ ] シークレット登録完了（LINE_CHANNEL_SECRET, LINE_CHANNEL_ACCESS_TOKEN）
- [ ] schema.sql 適用済み
- [ ] 全マイグレーション適用済み
- [ ] ビルド・デプロイ完了
- [ ] 管理ユーザー（staff_members）作成済み、APIキー取得済み
- [ ] Webhook URL 設定済み、検証成功
- [ ] LINE Login チャネル作成済み（同一プロバイダー内）
- [ ] LINE Login チャネルのステータスを「公開済み」に変更
- [ ] LIFF アプリ作成済み、LIFF_ID 取得済み
- [ ] 正しい VITE_LIFF_ID で再ビルド・再デプロイ完了
- [ ] 管理画面（apps/web）起動確認
- [ ] 友だち追加テスト成功
- [ ] メッセージ送信テスト成功

---

> **本マニュアルの最終更新**: 2026-04-09  
> **対象リポジトリ**: https://github.com/shudesu/line-harness-oss
