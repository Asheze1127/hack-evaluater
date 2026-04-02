# docs/AGENTS.md

AI向け `docs/` ナビゲーションガイド。現在の HackTrack 仕様に合わせて参照すること。

---

## プロダクト概要

**HackTrack** — ハッカソン運営向けのマルチテナント型スカウト管理基盤。

| 項目 | 内容 |
| --- | --- |
| 基本構造 | `Tenant > Hackathon` |
| 主なロール | `PlatformAdmin` / `TenantAdmin` / `HackathonOrganizer` / `Sponsor` / `Judge` / `Hacker` |
| コア機能 | 招待 URL + パスワード参加、スポンサー可視範囲制御、スカウト送信、メール通知 |
| 認証 | `dev=Magnito`、`prod=Cognito` |
| 実行環境 | `dev=Docker Compose`、`prod=AWS + EKS` |

---

## ファイルマップ

| ファイル | 内容の要点 | 参照すべき場面 |
| --- | --- | --- |
| [`specification.md`](./specification.md) | 仕様全体。課題、ロール、フロー、機能要件 | 仕様の背景と全体像を確認したいとき |
| [`details/01_feature-list_md.md`](./details/01_feature-list_md.md) | 機能一覧と優先度 | スコープ判断、P0/P1整理 |
| [`details/02_tech-stack.md`](./details/02_tech-stack.md) | 技術選定一覧。`dev/prod` のサービス切替を含む | 技術選定や依存追加の判断 |
| [`details/03_screen-flow_md.md`](./details/03_screen-flow_md.md) | 画面遷移 | 画面 / UX 設計、ルーティング実装 |
| [`details/04_permission-design.md`](./details/04_permission-design.md) | RBAC + ABAC 設計 | 認可実装、ミドルウェア設計 |
| [`details/05_erd.md`](./details/05_erd.md) | ERD とテーブル責務 | DB 設計、マイグレーション検討 |
| [`details/06_directory.md`](./details/06_directory.md) | `web / go-back / infra / docs` の配置方針 | 新規ファイルの置き場判断 |
| [`details/07_infrastructure.md`](./details/07_infrastructure.md) | `dev=Docker Compose`、`prod=AWS + EKS` のインフラ構成 | インフラ設計、デプロイ構成の確認 |
| [`details/08_logging.md`](./details/08_logging.md) | 構造化ログ、監査ログ、CloudWatch 方針 | ログ出力、監査証跡の設計 |

---

## 速読ポイント

- フロントは `feature / container` アーキテクチャ
- バックエンドは Go + Gin + OpenAPI + sqlc
- `APP_ENV=dev|prod` を主スイッチとして依存先を切り替える
- `prod` の API は EKS 上の Pod として動作し、HPA でオートスケーリングする
- `prod` の主要サービスは Route53、CloudFront、WAF、ALB、EKS、RDS、ElastiCache、Cognito、SES、S3、CloudWatch、Secrets Manager、ECR
