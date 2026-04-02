# 技術スタック

---

## 0️⃣ 前提定義

| 項目 | 内容 |
| --- | --- |
| プロダクト種別 | マルチテナント型の BtoB Web アプリ。ハッカソン運営向けのスカウト管理基盤 |
| 想定ユーザー規模 | MVP は少数テナント、1 テナントあたり数十〜数百ユーザー、ピーク同時接続は数十程度を想定 |
| 可用性目標 | MVP は 99.5% 程度を目標。まずは機能成立と運用容易性を優先 |
| セキュリティ要件 | メール認証必須、テナント分離、RBAC + ABAC、監査可能性、個人情報の最小保持 |
| パフォーマンス要件 | 通常 API は p95 500ms 前後、画面初期表示は 2 秒前後を目安。メール送信などは非同期化を前提 |
| チーム体制 | 少人数での MVP 開発を想定。学習コストよりも保守性と実装速度を重視 |
| リリース頻度 | 初期は週次または隔週リリースを想定 |
| 予算制約 | コスト重視。`dev` はローカル OSS、`prod` は AWS マネージドサービスを使い分ける |
| 既存資産 | `web/` に Next.js 16 + TypeScript のフロントエンドを配置予定。`prod` は静的エクスポートで配信し、バックエンドは Go 前提で新規整備予定 |

---

# 1️⃣ 技術スタック構成

---

## 🖥 フロントエンド

| 項目 | 採用技術 | 採用理由 | 評価観点 | 備考 |
| --- | --- | --- | --- | --- |
| 言語 | TypeScript | 型安全に UI と API 契約を扱いやすく、少人数開発でも保守しやすい | 型安全性 / 学習コスト / 保守性 | 配置予定の `web/` と整合 |
| フレームワーク | Next.js 16（App Router） | ルーティング、レイアウト、認証画面、管理画面を 1 つの基盤で扱いやすい | エコシステム / 静的エクスポート適性 / パフォーマンス | `prod` は `output: export` 前提で、runtime SSR と Route Handlers は使わない |
| UIライブラリ | Tailwind CSS + shadcn/ui | MVP で開発速度を出しつつ、Admin Dashboard と運営画面の一貫性を出しやすい | デザイン一貫性 / 開発速度 | Tailwind / shadcn.ui は導入予定（現時点では未配置） |
| 状態管理 | TanStack Query + React Context | Server State と認証・テナント文脈を分離しやすく、Redux ほど重くしない | スケール耐性 / Server State分離 | UI の局所状態は React 標準 state を優先 |
| フォーム管理 | React Hook Form + Zod | スカウト入力や管理画面設定のバリデーションを軽量に実装できる | パフォーマンス / バリデーション | Zod は API 入出力の型共有にも相性が良い |
| テスト | Vitest + Testing Library + Playwright | UI 単体、画面挙動、主要導線 E2E を段階的に担保できる | カバレッジ / 実行速度 | MVP は重要導線の E2E を最小構成で開始 |

---

## 🧠 バックエンド

| 項目 | 採用技術 | 採用理由 | 評価観点 | 備考 |
| --- | --- | --- | --- | --- |
| 言語 | Go | 並行処理、型安全、デプロイ容易性のバランスが良く、API/認可層との相性が良い | 生産性 / 型安全 / 採用市場 | 将来の権限制御拡張にも向く |
| 実行環境 | Go アプリケーション + Docker | `dev` では Docker Compose、`prod` では EKS 上のコンテナ実行へ寄せやすい | 成熟度 / エコシステム | ローカルと本番の差分を抑えやすい |
| フレームワーク | Gin | Go での採用実績が多く、ルーティング、ミドルウェア、JSON API の実装速度を出しやすい | 構造化 / 拡張性 | 認証・認可ミドルウェアの実装とも相性が良い |
| API方式 | REST + OpenAPI First | Web、Admin Dashboard、通知処理との境界が明確で、将来のクライアント追加にも対応しやすい | 型安全 / 柔軟性 / 過不足 | `oapi-codegen` と Gin の組み合わせを前提 |
| ORM / DBアクセス | `sqlc` + `pgx` + `goose` | ORM より SQL 主導で制御しやすく、PostgreSQL を活かしやすい | 型統合 / マイグレーション管理 | MVP は複雑なドメインでも追いやすい構成を優先 |
| 認証 | `APP_ENV` に応じて `dev=Magnito`、`prod=Cognito` を切り替える | メール認証を統一的に扱いやすい | セキュリティ / OAuth対応 | API 側では JWT 検証を行う |
| 非同期処理 | アプリ内非同期ジョブ（goroutine） | スカウト送信 API のレイテンシを抑えつつ、SQS 導入前でもメール送信を API 応答から切り離せる | レイテンシ / 実装容易性 / 耐障害性 | `scouts` の永続化後に mailer を非同期起動し、SQS は将来検討 |

---

## 🗄 データベース

| 項目 | 採用技術 | 採用理由 | 評価観点 | 備考 |
| --- | --- | --- | --- | --- |
| メインDB | PostgreSQL 15+ | テナント、ユーザー、スカウト、設定などのトランザクション整合性が重要で、`UNIQUE NULLS NOT DISTINCT` を使えるため | ACID / スケール性 | `dev` は Docker 上 PostgreSQL 15+、`prod` は RDS PostgreSQL 15+ Multi-AZ。`prod` では RLS 併用を前提 |
| キャッシュ | Redis / ElastiCache for Redis | 招待トークン TTL、セッション補助、参照負荷軽減に使いやすい | レイテンシ改善 / TTL管理 | `dev` は Redis、`prod` は ElastiCache |
| 検索 | PostgreSQL の全文検索 + `pg_trgm` | MVP のスカウト対象検索は外部検索基盤なしで十分対応可能 | 全文検索性能 | 専用検索エンジンは後回し |
| 分析基盤 | 未採用（MVP） | 初期はプロダクト分析より運用ログと監査ログを優先する | BI連携 / ETL容易性 | 必要なら Athena / BigQuery などを後日検討 |

---

## ☁ インフラ / DevOps

| 項目 | 採用技術 | 採用理由 | 評価観点 | 備考 |
| --- | --- | --- | --- | --- |
| ホスティング | Docker Compose（dev）/ AWS EKS + S3 + CloudFront（prod） | ローカル再現性と本番のスケーラブルな公開構成を両立しやすい | 再現性 / 説明しやすさ | `prod` では API は EKS、frontend は Next.js 静的エクスポートを S3 + CloudFront で配信 |
| コンテナ | Docker / Kubernetes | ローカルと `prod` の両方でコンテナ前提に統一しやすい | 再現性 / 可搬性 | `prod` の Pod は HPA 対象 |
| IaC | Terraform + Kubernetes manifest | AWS リソースとワークロード定義を責務分離して管理しやすい | 構成管理 / 再現性 | Terraform は AWS、K8s manifest は Pod / HPA |
| CI/CD | GitHub Actions | リポジトリ連携が容易で、lint / test / build / deploy を段階的に自動化できる | 自動化 / 安定性 | MVP は main への反映を起点に運用 |
| 監視 | stdout / Docker logs（dev）、CloudWatch + Container Insights（prod） | 環境に応じて過不足なく可観測性を持てる | 可観測性 / シンプルさ | `prod` では EKS メトリクスを追う |
| ログ管理 | 標準出力（dev）/ CloudWatch Logs + S3（prod） | 監査ログと運用ログを `prod` で保全しやすい | トレーサビリティ | 詳細は logging doc で管理 |
| CDN | CloudFront | 静的 frontend 配信と公開入口の統一に向く | 配信経路の明瞭さ / TLS終端 | `prod` で採用 |
| 入口保護 | WAF + ALB + ACM | API 入口の保護と TLS 管理を分かりやすく構成できる | セキュリティ / 公開経路の整理 | `prod` で採用 |
| シークレット管理 | Secrets Manager | DB / Redis / SES 関連の秘匿情報を安全に扱いやすい | セキュリティ / 運用性 | `prod` で採用 |
| コンテナレジストリ | Amazon ECR | EKS 配備用イメージ管理 | 配布容易性 / 権限管理 | `prod` で採用 |
| DNS | Route53 | ドメイン管理と CloudFront 連携 | 公開経路の整理 | `prod` で採用 |

---

## 🔐 セキュリティ

| 項目 | 方針 | 評価観点 |
| --- | --- | --- |
| 認証方式 | `APP_ENV` に応じて `dev=Magnito`、`prod=Cognito` を切り替える | 標準準拠 / 拡張性 |
| 認可 | アプリケーション側で RBAC + ABAC を実装し、テナント境界と送信権限を厳格に分離する | RBAC / ABAC可否 |
| DB 境界防御 | `prod` では PostgreSQL RLS を併用し、テナント境界の防御層を増やす | 多層防御 / 誤実装耐性 |
| 通信 | `prod` では Route53 / CloudFront / WAF / ALB / ACM による公開経路を前提とする | TLS強制 / 証明書管理 |
| データ保護 | `dev` は `.env` 管理、`prod` は Secrets Manager と AWS 側暗号化を前提とする | 暗号化 / マスキング |
| 脆弱性対策 | Dependabot、Trivy、`npm audit`、`govulncheck` を組み合わせて継続検査する | 自動検査 / パッチ管理 |

---

# 2️⃣ 環境別の使い分け

| 領域 | `APP_ENV=dev` | `APP_ENV=prod` |
| --- | --- | --- |
| 認証 | Magnito | Amazon Cognito |
| メール送信 | Mailpit | Amazon SES |
| オブジェクトストレージ | MinIO | Amazon S3 |
| メインDB | PostgreSQL on Docker | Amazon RDS PostgreSQL Multi-AZ |
| キャッシュ | Redis on Docker | Amazon ElastiCache for Redis |
| アプリ実行 | Docker Compose | EKS + Next.js static export frontend |
| 監視 / ログ | 標準出力、Docker logs | CloudWatch / CloudWatch Logs |

---

# 3️⃣ MVP の技術選定方針

- MVP では「スカウト送信」「メール通知」「テナント設定」「Admin Dashboard」を成立させることを最優先とする。
- そのため、外部検索基盤、専用キャッシュ、分析基盤は導入しない。
- `APP_ENV=dev|prod` に応じて依存先を切り替える前提で設計する。
- `dev` はできる限りローカル OSS で揃える。
- `prod` の frontend は静的エクスポート前提とし、SSR や Route Handlers は Go API 側へ寄せる。
- `prod` は EKS、Cognito、SES、S3、RDS、ElastiCache、CloudWatch、Secrets Manager を用いる。
- Pod は Kubernetes HPA で自動スケールする。
- 将来の AI 機能は MVP の必須スタックに含めず、必要になった時点で別途技術選定する。
