# ディレクトリ構成

---

## 0️⃣ 設計前提

| 項目 | 内容 |
| --- | --- |
| リポジトリ構成 | 単一リポジトリ内に `web / go-back / infra / docs` を置く構成 |
| フロントエンド方針 | Next.js App Router + `feature / container` アーキテクチャ |
| バックエンド方針 | Go + Gin + OpenAPI + sqlc を前提に、責務ごとに整理する |
| インフラ方針 | `APP_ENV=dev|prod` を前提に、ローカル実行環境と AWS デプロイ環境の両方を持つ |
| MVP方針 | まずは P0 に必要なディレクトリを明確にし、汎用的すぎる置き場を避ける |

補足:

- Admin Dashboard も別フロントではなく `web/` の中に含める。
- `adminWeb/` は現時点では採用しない前提とする。
- 現在ある `web/src/api`、`web/src/component`、`web/src/hooks`、`web/src/util` は暫定置き場であり、最終的には `features`、`shared`、`lib` に寄せる。

---

## 1️⃣ リポジトリ全体構成

```text
root/
├── web/                    # フロントエンド（Next.js）
├── go-back/                # バックエンド（Go）
├── infra/                  # Docker / Terraform / Kubernetes
├── docs/                   # 仕様、設計、運用ドキュメント
├── scripts/                # 補助スクリプト
├── .github/                # CI / GitHub設定
├── .env.example            # 環境変数サンプル
└── README.md
```

### 各ディレクトリの責務

| ディレクトリ | 役割 |
| --- | --- |
| `web/` | 画面、ルーティング、フロントの機能実装 |
| `go-back/` | API、認可、DB アクセス、メール送信 |
| `infra/` | ローカル起動、AWS リソース、Kubernetes ワークロードの構成管理 |
| `docs/` | 仕様、画面、権限、ERD、インフラ、ログ設計 |
| `scripts/` | 開発補助コマンド、初期化、コード生成補助 |

---

## 2️⃣ フロントエンド構成

### 2.1 基本方針

- 画面のルーティングは `src/app/` に置く
- 実機能は `src/features/` に機能単位で置く
- 各 feature の中に `components`、`containers`、`hooks` などを置く
- 複数 feature で共有されるものだけを `src/shared/` に出す
- フレームワーク依存やクライアント生成などの横断関心は `src/lib/` に置く

### 2.2 推奨構成

```text
web/
├── public/
├── src/
│   ├── app/                       # App Router
│   │   ├── (public)/             # 未ログイン導線
│   │   ├── (console)/            # ログイン後導線
│   │   ├── globals.css
│   │   ├── layout.tsx
│   │   └── page.tsx
│   ├── features/                  # 機能単位
│   ├── shared/                    # 複数feature共有
│   ├── lib/                       # APIクライアント、env、query設定
│   ├── providers/                 # QueryProvider等
│   ├── styles/                    # 補助スタイル、トークン
│   └── types/                     # 全体共有型
├── tests/
│   └── e2e/
├── package.json
└── tsconfig.json
```

### 2.3 `app/` の推奨ルート構成

```text
src/app/
├── (public)/
│   ├── login/
│   ├── signup/
│   ├── verify-email/
│   └── invite/
│       ├── hackathons/[token]/
│       └── sponsors/[token]/
├── (console)/
│   ├── app/
│   │   ├── switch-context/
│   │   ├── scouts/
│   │   ├── hacker/
│   │   └── judge/
│   ├── platform/
│   ├── tenant/
│   └── organizer/
├── layout.tsx
├── globals.css
└── page.tsx
```

ルール:

- `page.tsx` は薄く保つ
- `page.tsx` から feature の `container` を呼ぶ
- ページ固有のロジックを `app/` 直下に溜め込まない

### 2.4 `features/` の基本構造

```text
src/features/
├── auth/
├── invite/
├── context-switch/
├── scout/
├── hacker/
├── judge/
├── sponsor/
├── tenant-admin/
├── hackathon-organizer/
└── platform-admin/
```

各 feature は以下の形を基本とする。

```text
src/features/<feature-name>/
├── api/                  # feature専用API呼び出し
├── components/           # 純粋な表示コンポーネント
├── containers/           # 状態取得、イベント接続、画面組み立て
├── hooks/                # feature専用hooks
├── schemas/              # zod schema
├── types/                # feature専用型
├── utils/                # feature専用補助関数
├── constants/            # feature専用定数
└── index.ts              # 外部公開用エントリ
```

### 2.5 `container` の役割

`container` は route layer と presentational component の間に置く。

責務:

- API 呼び出し hooks の利用
- URL パラメータや search params の解決
- form state と submit 処理の接続
- 権限や loading / error の分岐
- `components/` への props 受け渡し

責務に含めないもの:

- 汎用 UI 部品の実装
- feature をまたぐ共通 API クライアントの定義
- 他 feature の内部状態を直接触ること

### 2.6 feature の具体例

```text
src/features/scout/
├── api/
│   ├── create-scout.ts
│   └── list-available-hackathons.ts
├── components/
│   ├── scout-form.tsx
│   ├── scout-confirm.tsx
│   └── scout-complete.tsx
├── containers/
│   ├── scout-create-container.tsx
│   ├── scout-confirm-container.tsx
│   └── scout-complete-container.tsx
├── hooks/
│   ├── use-scout-form.ts
│   └── use-send-scout.ts
├── schemas/
│   └── scout-form-schema.ts
├── types/
│   └── scout.ts
└── index.ts
```

### 2.7 `shared/` に置くもの

複数 feature から再利用されるものだけを置く。

```text
src/shared/
├── components/
│   ├── ui/               # Button, Input, Dialog など
│   ├── layout/           # Header, Sidebar など
│   └── feedback/         # Loading, Empty, ErrorView など
├── hooks/
│   ├── use-current-user.ts
│   └── use-active-context.ts
├── constants/
├── utils/
└── types/
```

ルール:

- まだ 1 feature でしか使っていないものは `shared` に出さない
- `shared` を汎用ゴミ箱にしない

### 2.8 `lib/` に置くもの

```text
src/lib/
├── api/
│   ├── client.ts
│   ├── fetcher.ts
│   └── error.ts
├── auth/
│   ├── session.ts
│   └── token.ts
├── env/
│   └── client-env.ts
├── query/
│   └── query-client.ts
└── logger/
```

`lib/` は framework / infra に近い責務を置く場所であり、業務機能は置かない。

### 2.9 現在の `web/src` からの整理方針

現状:

```text
web/src/
├── api
├── app
├── component
├── hooks
└── util
```

今後の寄せ先:

| 現在の置き場 | 寄せ先 |
| --- | --- |
| `src/api` | `src/lib/api` または `src/features/*/api` |
| `src/component` | `src/shared/components` または `src/features/*/components` |
| `src/hooks` | `src/shared/hooks` または `src/features/*/hooks` |
| `src/util` | `src/shared/utils` または `src/features/*/utils` |

---

## 3️⃣ バックエンド構成

### 3.1 基本方針

- `go-back/` を API 実装のルートにする
- `Gin + OpenAPI + sqlc` 前提で、HTTP、usecase、repository、認可、DB を明確に分ける
- ただし過度な DDD 分割はせず、MVP に必要な粒度で整理する

### 3.2 推奨構成

```text
go-back/
├── cmd/
│   └── api/
│       └── main.go
├── internal/
│   ├── auth/
│   ├── authorization/
│   ├── platform/
│   ├── tenant/
│   ├── hackathon/
│   ├── invite/
│   ├── membership/
│   ├── sponsor/
│   ├── scout/
│   ├── mail/
│   ├── shared/
│   └── config/
├── db/
│   ├── migrations/
│   ├── queries/         # sqlc用SQL
│   └── sqlc/            # 生成コード
├── openapi/
│   └── api.yaml
├── tests/
│   ├── integration/
│   └── fixtures/
├── go.mod
└── go.sum
```

### 3.3 `internal/<feature>` の基本構造

```text
internal/<feature>/
├── handler.go
├── service.go
├── repository.go
├── model.go
├── policy.go
└── dto.go
```

例:

- `auth` : JWT 検証、認証済みユーザー解決
- `authorization` : RBAC / ABAC 判定、active context 解決
- `invite` : 招待作成、再発行、無効化、参加処理
- `scout` : スカウト送信、メール通知連携

### 3.4 DB まわりの置き場

```text
db/
├── migrations/
├── queries/
│   ├── users.sql
│   ├── tenants.sql
│   ├── hackathons.sql
│   ├── invites.sql
│   └── scouts.sql
└── sqlc/
```

ルール:

- 生 SQL は `db/queries` に置く
- 生成コードは `db/sqlc` に隔離する
- migration と query を同じ場所に混ぜない

---

## 4️⃣ インフラ構成

`dev` と `prod` を分けて置く。`prod` は Terraform と Kubernetes manifest を併用する。

```text
infra/
├── docker/
│   ├── compose.yml
│   ├── web.Dockerfile
│   └── api.Dockerfile
├── terraform/
│   ├── modules/
│   └── environments/
│       ├── dev/
│       └── prod/
├── kubernetes/
│   ├── base/
│   ├── dev/
│   └── prod/
└── README.md
```

補足:

- ローカル起動の正本は `docker/compose.yml`
- AWS リソースの正本は `terraform/`
- Pod / HPA / Service などのワークロード定義は `kubernetes/`

---

## 5️⃣ ドキュメント構成

```text
docs/
├── specification.md
├── details/
│   ├── 01_feature-list_md.md
│   ├── 02_tech-stack.md
│   ├── 03_screen-flow_md.md
│   ├── 04_permission-design.md
│   ├── 05_erd.md
│   ├── 06_directory.md
│   ├── 07_infrastructure.md
│   └── 08_logging.md
└── AGENTS.md
```

役割:

- `specification.md` は仕様の親
- `details/` は詳細設計の分冊

---

## 6️⃣ テスト構成

### 6.1 フロントエンド

```text
web/
├── src/features/<feature>/components/*.test.tsx
├── src/features/<feature>/hooks/*.test.ts
└── tests/e2e/
```

方針:

- unit test は feature 近傍に置く
- E2E は `web/tests/e2e` にまとめる

### 6.2 バックエンド

```text
go-back/
├── internal/<feature>/*_test.go
└── tests/
    ├── integration/
    └── fixtures/
```

方針:

- 単体テストは package 近傍
- DB や HTTP をまたぐものは `tests/integration`

---

## 7️⃣ この構成で守りたいルール

- フロントの機能実装は必ず `features` 配下から始める
- `container` は feature ごとに持つ
- 共有化は早すぎず、必要になってから `shared` に上げる
- `app/` は route、`features/` は機能、`shared/` は共通、`lib/` は基盤と役割を分ける
- バックエンドは `internal` 配下を機能ごとに分けつつ、DB 生成コードと migration を分離する
- `infra` はローカルとデプロイを混ぜない

---

## 8️⃣ 避けたいアンチパターン

- `components/` や `utils/` を root に大きく育てて何でも入れる
- 1 つの feature でしか使わないものを最初から `shared` に置く
- `page.tsx` に API 呼び出しや複雑な form 制御を書く
- バックエンドで HTTP、DB、認可ロジックを 1 ファイルに混ぜる
- `infra/` に使っていない基盤を先回りで増やす
