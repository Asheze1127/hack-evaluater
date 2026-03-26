# docs/AGENTS.md

AI向け `docs/` ナビゲーションガイド。ここを起点に必要なドキュメントを特定して参照すること。

---

## プロダクト概要

**Hackathon Board** — ハッカソン向けWebダッシュボード。

| 対象ユーザー | ハッカソン参加者（初心者）/ メンター / 運営 |
| --- | --- |
| コア機能 | 進捗可視化（`/progress`）、Q&A AI一次回答（`/question`）、SlackからGitHub Issue自動起票 |
| MVP優先度 | P0のみまず実装。P1以降は体験向上フェーズ |

---

## ファイルマップ

| ファイル | 内容の要点 | 参照すべき場面 |
| --- | --- | --- |
| [`specification.md`](./specification.md) | Why / What / How の全体概要。課題・スコープ・アイデア・開発ステップ | 仕様の背景・意図を確認したいとき |
| [`details/01_feature-list.md`](./details/01_feature-list.md) | 機能一覧と優先度（P0〜P3）。カテゴリA（進捗）・B（Q&A）・C（Issue化）・D（分析） | 機能追加・スコープ判断・優先度確認 |
| [`details/02_tech-stack.md`](./details/02_tech-stack.md) | 技術選定一覧（フロント: Next.js/TS/ShadCN、バック: Go/api-codegen/sqlc、DB: PostgreSQL、インフラ: AWS ECS/CDK/SQS）| 技術選定理由・依存追加の判断 |
| [`details/03_screen-flow.md`](./details/03_screen-flow.md) | Web画面遷移（S-01〜S-05）とSlack操作遷移（L-01〜L-05）。Mermaidフローチャートあり | 画面・UX設計・遷移実装 |
| [`details/04_permission-design.md`](./details/04_permission-design.md) | RBAC設計。MVPでは `MENTOR` ロールのみ。ABAC・チームスコープは将来対応 | 認可実装・ミドルウェア設計 |
| [`details/05_erd.md`](./details/05_erd.md) | データベース設計 | データベースの設計、マイグレーションをするとき |
| [`details/06_directory.md`](./details/06_directory.md) | Monorepoディレクトリ構成（`backend/` Go、`web/` Next.js、`lambda/`、`infra/terraform/`）| ファイル配置・新規ファイル作成場所の確認 |
| [`details/07_infrastructure.md`](./details/07_infrastructure.md) | AWSインフラ構成（ECS、CloudFront/WAF、SQS、RDB）。Mermaidアーキテクチャ図あり | インフラ変更・デプロイ設計 |
| [`details/08_logging.md`](./details/08_logging.md) | 保存するべきログの保存内容 | バックエンドでログ保存をするとき |

---

## アーキテクチャ要点（素早い把握用）

```
Web (/slide, /get)
  └→ POST /slide [署名検証 → 重複チェック → DB保存 → 200 OK]
```

- LLM: Amazon Bedrock（Vercel AI SDK + `@ai-sdk/amazon-bedrock` アダプター）devローカル → geminiFlash
- 認証: Webは独自Session認証
