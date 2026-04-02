# ログ設計

---

## 0️⃣ 設計前提

| 項目 | 内容 |
| --- | --- |
| 対象システム | Next.js フロントエンド、Go API、メール送信処理 |
| ログ方式 | 構造化ログ（JSON）を標準とする |
| 主保管先 | `APP_ENV=dev` は標準出力中心、`APP_ENV=prod` は CloudWatch Logs を主保管先とする |
| アーカイブ | `APP_ENV=prod` では S3 へ長期保管する |
| 相関 ID | `trace_id`、`request_id` を必須とする |
| 権限文脈 | `active_context_role`、`active_context_scope_type`、`active_context_scope_id` を記録する |
| 個人情報方針 | パスワード、招待トークン、JWT は出力禁止。メールアドレスは原則マスクまたはハッシュで扱う |
| 監査方針 | 設定変更、権限付与、招待操作、スカウト送信は必ず監査可能にする |

補足:

- ログ本体は ERD ではなく、可観測性基盤で保持する。
- `PlatformAdmin` / `TenantAdmin` / `HackathonOrganizer` / `Sponsor` / `Judge` / `Hacker` のロール文脈を必ず追えるようにする。

---

## 1️⃣ ログの目的

- 障害解析をしやすくする
- テナント境界や権限制御の誤りを追跡できるようにする
- 招待、ロール付与、スカウト送信の監査証跡を残す
- メール送信失敗を検知しやすくする
- MVP の主要 KPI を後から集計できるようにする

---

## 2️⃣ ログ分類

| 種別 | 目的 | 主な利用者 | 保持目安 |
| --- | --- | --- | --- |
| Application Log | アプリケーションの動作確認、障害解析 | 開発 / 運用 | 30日 |
| Access Log | HTTP リクエスト追跡、性能確認 | 運用 | 90日 |
| Authorization Log | RBAC / ABAC の評価結果追跡 | 開発 / セキュリティ | 90日 |
| Audit Log | 設定変更、権限付与、招待操作の証跡 | セキュリティ / 運営 | 1年 |
| Security Log | ログイン失敗、異常アクセス、連続 deny の検知 | セキュリティ | 1年 |
| Business Log | Tenant 作成、参加完了、スカウト送信など KPI 集計 | PM / Biz | 180日 |
| Mail Delivery Log | スカウト通知メールの送達結果追跡 | 運用 | 180日 |

---

## 3️⃣ 環境別の出力先

| 環境 | 出力先 | 備考 |
| --- | --- | --- |
| `APP_ENV=dev` | アプリ標準出力、Docker ログ | まずはローカル確認を優先 |
| `APP_ENV=prod` | CloudWatch Logs + S3 Archive | 長期保管と監査対応を優先 |

推奨ロググループ例:

```text
/hacktrack/{env}/application
/hacktrack/{env}/access
/hacktrack/{env}/authorization
/hacktrack/{env}/audit
/hacktrack/{env}/security
/hacktrack/{env}/business
/hacktrack/{env}/mail
```

### 3.1 フロントエンドのログ方針

- `APP_ENV=dev` では、ブラウザ console と React の開発エラー表示を主な確認手段とする。
- `APP_ENV=prod` では、クライアントの全操作ログは収集しない。
- `APP_ENV=prod` で収集するのは、未捕捉例外、`unhandledrejection`、初期表示を阻害する致命的な API エラーなどの障害解析に必要な最小限の `web.client_error` イベントに限定する。
- フロントエンドから送るエラーログにも `trace_id` と route 情報を付与し、API 側の構造化ログに集約する。

---

## 4️⃣ 共通フィールド

### 4.1 全ログに共通で持たせる項目

| フィールド | 必須 | 説明 |
| --- | --- | --- |
| `timestamp` | 必須 | ISO8601 UTC 形式 |
| `level` | 必須 | `DEBUG` / `INFO` / `WARN` / `ERROR` |
| `service` | 必須 | `web` / `api` / `mailer` など |
| `environment` | 必須 | `dev` / `prod` |
| `cluster_name` | 条件付き | `prod` の EKS クラスタ名 |
| `namespace` | 条件付き | `prod` の Kubernetes Namespace |
| `pod_name` | 条件付き | `prod` の Pod 名 |
| `trace_id` | 必須 | リクエスト横断の相関 ID |
| `request_id` | 条件付き | HTTP リクエスト単位の ID |
| `event_name` | 必須 | `tenant.created` などのイベント識別子 |
| `event_category` | 必須 | `application` / `audit` / `security` など |
| `message` | 必須 | 人間が読める短い説明 |
| `metadata` | 任意 | 補助情報 |

### 4.2 認証・認可・監査で共通で持たせる項目

| フィールド | 必須 | 説明 |
| --- | --- | --- |
| `actor_user_id` | 推奨 | 操作主体のユーザー ID |
| `actor_role_bindings` | 推奨 | 評価時に持っていた binding の要約 |
| `active_context_role` | 推奨 | 現在利用中のロール |
| `active_context_scope_type` | 推奨 | `global` / `tenant` / `hackathon` |
| `active_context_scope_id` | 推奨 | 現在のスコープ ID |
| `tenant_id` | 推奨 | 対象 Tenant |
| `hackathon_id` | 推奨 | 対象 Hackathon |
| `resource_type` | 推奨 | `tenant` / `hackathon` / `invite` / `scout` など |
| `resource_id` | 推奨 | 対象リソース ID |
| `action` | 推奨 | `create` / `update` / `send` / `revoke` など |
| `outcome` | 推奨 | `success` / `deny` / `error` |
| `reason` | 任意 | deny / error の理由 |

### 4.3 HTTP アクセスで持たせる項目

| フィールド | 必須 | 説明 |
| --- | --- | --- |
| `http_method` | 必須 | `GET` / `POST` など |
| `http_path` | 必須 | ルートパターン基準で記録 |
| `status_code` | 必須 | HTTP ステータスコード |
| `latency_ms` | 必須 | レスポンス時間 |
| `client_ip` | 推奨 | セキュリティ用途。一般ログではマスク可 |
| `user_agent` | 推奨 | クライアント識別 |

---

## 5️⃣ ログレベル方針

| レベル | 用途 |
| --- | --- |
| `DEBUG` | 開発時の詳細追跡。`prod` では原則無効 |
| `INFO` | 正常系イベント、主要な業務イベント |
| `WARN` | 想定内だが注意が必要な異常。期限切れ招待、権限不足、メール再送など |
| `ERROR` | API 失敗、送信失敗、DB エラーなど処理失敗 |

運用ルール:

- `prod` では `DEBUG` を無効または強くサンプリングする。
- `Authorization Log` の deny は `WARN`、予期しない認可失敗は `ERROR` とする。

---

## 6️⃣ P0 で必ず出すイベント

| event_name | 種別 | レベル | 説明 |
| --- | --- | --- | --- |
| `auth.login.succeeded` | Security | INFO | ログイン成功 |
| `auth.login.failed` | Security | WARN | ログイン失敗 |
| `tenant.created` | Audit / Business | INFO | Tenant 作成 |
| `tenant.status.updated` | Audit | INFO | Tenant 状態変更 |
| `tenant_admin.assigned` | Audit | INFO | 初期 `TenantAdmin` 付与 |
| `hackathon.created` | Audit / Business | INFO | Hackathon 作成 |
| `hackathon_organizer.assigned` | Audit | INFO | `HackathonOrganizer` 付与 |
| `invite.created` | Audit | INFO | 招待 URL 発行 |
| `invite.reissued` | Audit | INFO | 招待再発行 |
| `invite.revoked` | Audit | WARN | 招待無効化 |
| `invite.accepted` | Audit / Business | INFO | 招待参加成功 |
| `invite.accept.failed` | Security | WARN | 期限切れ、パスワード不一致、無効化済みなど |
| `sponsor_scope.updated` | Audit | INFO | Sponsor 表示範囲変更 |
| `hackathon.scout_enabled.updated` | Audit | INFO | スカウト ON / OFF 変更 |
| `hackathon.judge_send_enabled.updated` | Audit | INFO | Judge 送信権限変更 |
| `authz.denied` | Authorization / Security | WARN | 認可拒否 |
| `scout.sent` | Audit / Business | INFO | スカウト送信成功 |
| `scout.send.failed` | Application | ERROR | スカウト送信失敗 |
| `mail.scout_delivery.succeeded` | Mail Delivery | INFO | 通知メール送達成功 |
| `mail.scout_delivery.failed` | Mail Delivery / Security | ERROR | 通知メール送達失敗 |

---

## 7️⃣ ログ種別ごとの設計

### 7.1 Application Log

用途:

- 例外、DB エラー、外部サービスエラーの追跡
- バッチや送信処理の失敗原因調査

例:

```json
{
  "timestamp": "2026-03-27T13:00:00Z",
  "level": "ERROR",
  "service": "api",
  "environment": "prod",
  "trace_id": "trc_01",
  "request_id": "req_01",
  "event_name": "scout.send.failed",
  "event_category": "application",
  "actor_user_id": "usr_01",
  "tenant_id": "ten_01",
  "hackathon_id": "hack_01",
  "resource_type": "scout",
  "resource_id": "sct_01",
  "message": "Scout send transaction failed",
  "reason": "db_write_failed",
  "metadata": {
    "error_code": "DB_TX_ROLLBACK"
  }
}
```

### 7.2 Access Log

用途:

- API 利用状況と性能把握
- 403 / 404 / 500 の偏り確認

例:

```json
{
  "timestamp": "2026-03-27T13:00:00Z",
  "level": "INFO",
  "service": "api",
  "environment": "prod",
  "trace_id": "trc_02",
  "request_id": "req_02",
  "event_name": "http.request.completed",
  "event_category": "access",
  "actor_user_id": "usr_02",
  "active_context_role": "tenant_admin",
  "active_context_scope_type": "tenant",
  "active_context_scope_id": "ten_01",
  "tenant_id": "ten_01",
  "http_method": "POST",
  "http_path": "/tenant/hackathons",
  "status_code": 201,
  "latency_ms": 85,
  "message": "Request completed"
}
```

### 7.3 Authorization Log

用途:

- RBAC / ABAC の deny 理由追跡
- 意図しない権限漏れの検知

運用ルール:

- deny は全件記録する
- allow は高リスク操作だけ記録する

例:

```json
{
  "timestamp": "2026-03-27T13:00:00Z",
  "level": "WARN",
  "service": "api",
  "environment": "prod",
  "trace_id": "trc_03",
  "request_id": "req_03",
  "event_name": "authz.denied",
  "event_category": "authorization",
  "actor_user_id": "usr_03",
  "actor_role_bindings": [
    "tenant_admin:tenant:ten_01",
    "sponsor:tenant:ten_01"
  ],
  "active_context_role": "sponsor",
  "active_context_scope_type": "tenant",
  "active_context_scope_id": "ten_01",
  "tenant_id": "ten_01",
  "hackathon_id": "hack_99",
  "resource_type": "hackathon",
  "resource_id": "hack_99",
  "action": "read",
  "outcome": "deny",
  "reason": "sponsor_scope_denied",
  "message": "Access denied by sponsor visibility scope"
}
```

### 7.4 Audit Log

用途:

- 設定変更と権限変更の証跡保持
- 誰がいつ何を変えたかの説明責任

対象:

- Tenant 作成 / 状態変更
- `TenantAdmin` / `HackathonOrganizer` 付与
- 招待発行 / 再発行 / 無効化
- Sponsor 表示範囲変更
- Hackathon のスカウト ON / OFF
- Judge 送信権限変更
- スカウト送信

例:

```json
{
  "timestamp": "2026-03-27T13:00:00Z",
  "level": "INFO",
  "service": "api",
  "environment": "prod",
  "trace_id": "trc_04",
  "request_id": "req_04",
  "event_name": "invite.reissued",
  "event_category": "audit",
  "actor_user_id": "usr_04",
  "active_context_role": "hackathon_organizer",
  "active_context_scope_type": "hackathon",
  "active_context_scope_id": "hack_01",
  "tenant_id": "ten_01",
  "hackathon_id": "hack_01",
  "resource_type": "invite",
  "resource_id": "inv_02",
  "action": "reissue",
  "outcome": "success",
  "message": "Invite reissued",
  "metadata": {
    "previous_invite_id": "inv_01",
    "target_role": "hacker",
    "expires_in_days": 14
  }
}
```

### 7.5 Security Log

用途:

- 不正アクセスや異常傾向の検知
- 認証・招待悪用の兆候把握

最低限記録するもの:

- ログイン失敗
- 招待パスワード不一致の連続発生
- 期限切れ / 無効化済み招待への継続アクセス
- 短時間での多量 `authz.denied`
- 連続 401 / 403 / 404 の偏り

### 7.6 Business Log

用途:

- KPI 集計
- MVP の利用状況把握

主要イベント:

- `tenant.created`
- `hackathon.created`
- `invite.accepted`
- `scout.sent`

注意:

- Business Log では個人情報を直接持たず、原則 `user_id` と `tenant_id` で集計する。

### 7.7 Mail Delivery Log

用途:

- SES / Mailpit 送信成否の確認
- スカウト通知失敗時の再調査

例:

```json
{
  "timestamp": "2026-03-27T13:00:00Z",
  "level": "ERROR",
  "service": "mailer",
  "environment": "prod",
  "trace_id": "trc_05",
  "event_name": "mail.scout_delivery.failed",
  "event_category": "mail",
  "tenant_id": "ten_01",
  "hackathon_id": "hack_01",
  "resource_type": "scout",
  "resource_id": "sct_01",
  "outcome": "error",
  "reason": "ses_throttling",
  "message": "Scout delivery failed",
  "metadata": {
    "provider": "ses",
    "provider_message_id": "ses_msg_01",
    "recipient_email_hash": "sha256:xxxxx"
  }
}
```

---

## 8️⃣ マスキング方針

### 絶対に出さないもの

- 招待 URL の生トークン
- 招待パスワードの平文
- JWT / リフレッシュトークン
- Authorization ヘッダ
- SES / Cognito のシークレット

### 原則マスクまたはハッシュで出すもの

| 対象 | 方針 |
| --- | --- |
| メールアドレス | 原則ハッシュ化。どうしても必要なときだけ部分マスク |
| IP アドレス | Security Log では保持可。その他では必要に応じて匿名化 |
| User-Agent | 長すぎる場合はトリム |
| 入力本文 | スカウト本文の全文は Application Log に出さない |

---

## 9️⃣ 保持ポリシー

| 種別 | 保持期間 | 理由 |
| --- | --- | --- |
| Application | 30日 | 障害解析の即時性を優先 |
| Access | 90日 | 性能傾向とアクセス調査 |
| Authorization | 90日 | 権限不具合の追跡 |
| Audit | 1年 | 監査証跡の保持 |
| Security | 1年 | セキュリティ調査 |
| Business | 180日 | MVP の利用分析 |
| Mail Delivery | 180日 | 通知失敗追跡 |

補足:

- 長期保持が必要なログは S3 にアーカイブする。
- `prod` では圧縮・ライフサイクル管理を前提とする。

---

## 🔟 アラート方針

P0 で最低限ほしいアラート:

- 5xx が一定閾値を超えたとき
- `auth.login.failed` が短時間に急増したとき
- `invite.accept.failed` が短時間に急増したとき
- `authz.denied` が特定エンドポイントで急増したとき
- `mail.scout_delivery.failed` が連続したとき
- ログ基盤への出力停止、または急減が起きたとき

---

## 1️⃣1️⃣ 実装ルール

### 11.1 ミドルウェアで付与するもの

- `trace_id`
- `request_id`
- `actor_user_id`
- `active_context_role`
- `active_context_scope_type`
- `active_context_scope_id`
- `tenant_id`
- `hackathon_id`

詳細方針:

- `trace_id` の正本は API ingress ミドルウェアとし、`X-Trace-Id` ヘッダーが妥当なら引き継ぎ、存在しなければ API 側で新規発行する。
- API はレスポンスヘッダーにも `X-Trace-Id` を返し、Web は以降の API 呼び出しや `web.client_error` 送信時に同じ値を再利用する。
- API から mailer へ処理を引き渡すときは、同じ `trace_id` を Go の `context.Context` に載せて伝播する。
- 将来 mailer が別プロセスやキューを経由する場合も、同じ `trace_id` をヘッダーまたはメッセージ属性に積む。

### 11.2 監査ログの出し分け

- 状態変更 API は、DB コミット成功後に `Audit Log` を出す
- 認可 deny は、更新失敗でも `Authorization Log` として残す
- `scout.sent` は `Audit Log` と `Business Log` の両方に出してよい

### 11.3 冪等性に関する考え方

- 同一操作の再送に備え、`request_id` またはアプリ側冪等キーで重複判定しやすくする
- 監査ログには重複判定結果も `metadata` で残せるようにする

### 11.4 フロントエンド起点の障害ログ

- `web.client_error` は `service=web`、`event_category=application` で記録する。
- 対象は未捕捉例外、`unhandledrejection`、初期表示失敗、主要 API の致命的エラーに限定する。
- UI 操作の逐次イベントや全文入力内容は送らない。

---

## 1️⃣2️⃣ このドキュメントの対象外

- Datadog など外部 SaaS 監視基盤の導入判断
- OpenTelemetry の詳細配線
- 将来の SIEM 連携設計

MVP では、CloudWatch Logs を中心にした構造化ログ運用で十分とする。
