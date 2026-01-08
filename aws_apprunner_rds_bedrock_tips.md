# AWS App Runner + RDS PostgreSQL + Bedrock デプロイガイド

**作成日**: 2026年1月4日
**対象**: Rails API Backend + Next.js Frontend

---

## アーキテクチャ概要

```
┌─────────────┐     ┌──────────────┐     ┌─────────────┐
│   Vercel    │────▶│  App Runner  │────▶│     RDS     │
│  (Frontend) │     │  (Backend)   │     │ (PostgreSQL)│
└─────────────┘     └──────┬───────┘     └─────────────┘
                          │
                          ▼
                   ┌─────────────┐
                   │   Bedrock   │
                   │   (Claude)  │
                   └─────────────┘
```

---

## 1. RDS PostgreSQL セットアップ

### 1.1 RDS作成時の設定
- **Engine**: PostgreSQL 15+
- **Template**: Free tier（開発用）/ Production（本番用）
- **Public access**: Yes（開発時）/ No（VPC内からのみアクセス）
- **VPC Security Group**: App Runnerからのアクセスを許可

### 1.2 セキュリティグループ設定
```
Inbound Rules:
- PostgreSQL (5432) from 0.0.0.0/0（開発時のみ）
- または App Runner VPC Connectorからのみ許可（本番推奨）
```

### 1.3 マイグレーション実行
```bash
# ローカルからRDSに接続してマイグレーション
DATABASE_URL=postgres://user:password@host:5432/dbname bundle exec rails db:migrate
DATABASE_URL=postgres://user:password@host:5432/dbname bundle exec rails db:seed
```

---

## 2. App Runner セットアップ

### 2.1 ECRリポジトリ作成
```bash
aws ecr create-repository --repository-name chronos-backend --region eu-central-1
```

### 2.2 Dockerイメージのビルド＆プッシュ
```bash
# ECRログイン
aws ecr get-login-password --region eu-central-1 | \
  docker login --username AWS --password-stdin <ACCOUNT_ID>.dkr.ecr.eu-central-1.amazonaws.com

# ARM Mac用: linux/amd64でビルド
docker build --platform linux/amd64 -t app-backend -f docker/Dockerfile.backend backend/

# タグ付け
docker tag app-backend:latest <ACCOUNT_ID>.dkr.ecr.eu-central-1.amazonaws.com/chronos-backend:latest

# プッシュ
docker push <ACCOUNT_ID>.dkr.ecr.eu-central-1.amazonaws.com/chronos-backend:latest
```

### 2.3 App Runner サービス作成
- **Source**: Container registry (Amazon ECR)
- **Deployment trigger**: Manual or Automatic
- **Port**: 8080（Railsのデフォルト変更推奨）
- **Health check path**: /health

### 2.4 環境変数
```
DATABASE_URL=postgres://user:password@rds-endpoint:5432/dbname
RAILS_ENV=production
RAILS_LOG_TO_STDOUT=1
HTTP_PORT=8080
LLM_PROVIDER=bedrock
AWS_REGION=eu-central-1
AWS_BEDROCK_MODEL_ID=eu.anthropic.claude-sonnet-4-5-20250929-v1:0
```

---

## 3. Bedrock 設定

### 3.1 IAMロール作成（App Runner用）

#### Trust Policy
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "tasks.apprunner.amazonaws.com"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
```

#### Permission Policy
`AmazonBedrockFullAccess` をアタッチ

または最小権限:
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "bedrock:InvokeModel",
        "bedrock:InvokeModelWithResponseStream"
      ],
      "Resource": "*"
    }
  ]
}
```

### 3.2 App Runnerへのロール設定
1. App Runner → Service → Configuration → Edit
2. Security → Instance role で作成したロールを選択
3. Save changes（自動でデプロイが開始）

### 3.3 モデルID（重要！）

**Claude Sonnet 4.5（3.5 v2）はCross-region inference profileが必要**

❌ 間違い:
```
anthropic.claude-sonnet-4-5-20250929-v1:0
```

✅ 正しい（リージョンプレフィックス付き）:
```
eu.anthropic.claude-sonnet-4-5-20250929-v1:0  # Europe
us.anthropic.claude-sonnet-4-5-20250929-v1:0  # US
```

---

## 4. CORS 設定

### Rails側 (config/initializers/cors.rb)
```ruby
Rails.application.config.middleware.insert_before 0, Rack::Cors do
  allow do
    origins "http://localhost:3000",
            "https://your-app.vercel.app"

    resource "*",
      headers: :any,
      methods: [:get, :post, :put, :patch, :delete, :options, :head],
      expose: ["X-User-ID"]
  end
end
```

---

## 5. デバッグ方法

### 5.1 CloudWatch Logs

#### ログの場所
```
CloudWatch → Log groups → /aws/apprunner/<service-name>/application
```

#### Live Tail（リアルタイムログ）
1. CloudWatch → Live Tail
2. Log groups で該当サービスを選択
3. Start をクリック
4. 別タブでアプリを操作してエラーを確認

#### フィルター検索
```
# エラーのみ表示
error

# 特定のキーワード
AccessDenied
Bedrock
500
```

### 5.2 よくあるエラーと解決策

#### エラー: `Zeitwerk::NameError`
```
expected file /rails/app/services/llm/openrouter_provider.rb to define constant Llm::OpenrouterProvider
```
**原因**: ファイル名とクラス名の不一致
**解決**: ファイル名をクラス名に合わせてリネーム
- `openrouter_provider.rb` → `open_router_provider.rb`

#### エラー: `CORS policy blocked`
**原因**: バックエンドでフロントエンドのオリジンが許可されていない
**解決**: `cors.rb`にVercelのドメインを追加

#### エラー: `on-demand throughput isn't supported`
```
Invocation of model ID anthropic.claude-sonnet-4-5-20250929-v1:0 with on-demand throughput isn't supported
```
**原因**: モデルがCross-region inference profile必須
**解決**: モデルIDを`eu.anthropic.claude-sonnet-4-5-20250929-v1:0`に変更

#### エラー: `AccessDeniedException`
**原因**: IAMロールにBedrock権限がない
**解決**: 
1. IAMロールに`AmazonBedrockFullAccess`を追加
2. App RunnerのInstance roleに設定

#### エラー: `connection refused` (Health check)
```
dial tcp 127.0.0.1:3000: connect: connection refused
```
**原因**: デプロイ中の一時的なエラー（通常は問題なし）
**確認**: デプロイ完了後にStatusがRunningになっていればOK

### 5.3 ブラウザ側デバッグ

#### Network タブ
1. Developer Tools (F12) → Network
2. XHRフィルター
3. リクエストのStatus Codeを確認
4. Response内容を確認

---

## 6. デプロイチェックリスト

### 初回デプロイ
- [ ] RDS作成＆セキュリティグループ設定
- [ ] ECRリポジトリ作成
- [ ] Dockerイメージビルド＆プッシュ
- [ ] App Runnerサービス作成
- [ ] 環境変数設定
- [ ] IAMロール作成＆設定
- [ ] RDSマイグレーション実行
- [ ] Vercelにフロントエンドデプロイ
- [ ] CORS設定
- [ ] 動作確認

### 更新デプロイ
```bash
# 1. Dockerイメージ再ビルド
docker build --platform linux/amd64 -t app-backend -f docker/Dockerfile.backend backend/

# 2. タグ付け＆プッシュ
docker tag app-backend:latest <ACCOUNT_ID>.dkr.ecr.eu-central-1.amazonaws.com/chronos-backend:latest
docker push <ACCOUNT_ID>.dkr.ecr.eu-central-1.amazonaws.com/chronos-backend:latest

# 3. App Runnerでデプロイ
# Console → Deploy ボタンをクリック
# または自動デプロイが有効なら自動で開始
```

### マイグレーションがある場合
```bash
# デプロイ前にマイグレーション実行
DATABASE_URL=postgres://... bundle exec rails db:migrate
```

---

## 7. コスト最適化

### App Runner
- 開発時: 最小インスタンス数を1に設定
- アイドル時の自動スケールダウン有効化

### RDS
- 開発時: db.t3.micro（Free tier対象）
- 本番時: 必要に応じてスケールアップ

### Bedrock
- 使用量ベースの課金
- Claude Sonnet 4.5: 入力$3/1M tokens, 出力$15/1M tokens

---

## 8. セキュリティベストプラクティス

1. **RDSをパブリックアクセス無効に**: VPC Connectorを使用
2. **環境変数にシークレット**: Secrets Managerの使用を検討
3. **IAM最小権限の原則**: Bedrock呼び出しに必要な権限のみ
4. **CORS制限**: 特定のオリジンのみ許可
5. **HTTPS強制**: App RunnerとVercelはデフォルトでHTTPS

---

## 参考リンク

- [AWS App Runner Documentation](https://docs.aws.amazon.com/apprunner/)
- [Amazon Bedrock Model IDs](https://docs.aws.amazon.com/bedrock/latest/userguide/model-ids.html)
- [Cross-region inference](https://docs.aws.amazon.com/bedrock/latest/userguide/cross-region-inference.html)
- [Rails on App Runner](https://aws.amazon.com/blogs/containers/deploy-rails-applications-to-aws-app-runner/)



