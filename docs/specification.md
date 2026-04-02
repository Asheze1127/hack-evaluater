# 仕様整理：HackTrack テナント・ハッカソン参加管理とダイレクトスカウト機能

## 1. 概要
HackTrack を、テナント配下に複数のハッカソンを持てるマルチイベント型プラットフォームとして整理する。  
権限は `PlatformAdmin`、`TenantAdmin`、`HackathonOrganizer`、`Sponsor`、`Judge`、`Hacker` に分け、各ロールはスコープ付きで付与する。  
同一ユーザーが複数ロールを持つことを許容し、たとえば 1 人のユーザーが同じ Tenant で `TenantAdmin` と `HackathonOrganizer` の両方を持つ状態を前提にする。

本機能の中心価値は以下の 3 点である。

- ハッカソンごとの参加者・審査員・スポンサーの参加管理を、手動登録ではなく招待導線で回せること
- Tenant と Hackathon の責任範囲を分け、運営権限を適切に委譲できること
- アプリ内スカウト送信とメール通知を、テナント / ハッカソン境界を守りながら安全に行えること

本ドキュメントでは、機能仕様と制約条件のみを扱う。

## 2. 解決したい課題
### 2.1. 現状の課題
- ハッカソンごとに参加者や審査員を手動で追加する運用負荷が高い。
- 同一テナント内で複数回ハッカソンを開催する場合、イベント単位の切り分けがしづらい。
- 開催団体全体の管理者と、各ハッカソンの運営担当者の責任範囲を分けたい。
- スポンサーはテナントに紐づく一方、実際に見せたいハッカソン範囲は限定したい。
- スカウト送信をアプリ内で完結させつつ、出場者にはメールで確実に届けたい。

### 2.2. 対象ユーザー
- `PlatformAdmin`: プロバイダー側の最小グローバル管理者。Tenant の作成と初期セットアップを担う。
- `TenantAdmin`: 開催団体側の Tenant 管理者。配下 Hackathon や Sponsor 関連設定を管理する。
- `HackathonOrganizer`: 各 Hackathon の運営担当者。招待管理や Hackathon 単位設定を行う。
- `Sponsor`: Tenant に紐づく協賛企業担当者。許可された Hackathon 範囲で閲覧・スカウト送信を行う。
- `Judge`: Hackathon 審査員。Hackathon に参加し、必要に応じてスカウト送信権限を持つ。
- `Hacker`: Hackathon 参加者。スカウト受信可否を設定し、スカウトをメールで受け取る。

### 2.3. 成功条件
- `PlatformAdmin` が Tenant を作成し、初期 `TenantAdmin` を付与できる。
- `TenantAdmin` が Tenant 配下に Hackathon を作成できる。
- `TenantAdmin` が `HackathonOrganizer` を各 Hackathon に割り当てられる。
- `Hacker` と `Judge` が Hackathon 参加用 URL と参加パスワードで参加できる。
- `Sponsor` が Tenant 参加用 URL と参加パスワードで参加できる。
- 招待 URL の有効期限を `5 / 7 / 14 / 30 / 60 / 90日` から選択できる。
- 招待 URL を再発行または無効化できる。
- `Sponsor` は `TenantAdmin` が許可した Hackathon だけを閲覧できる。
- `HackathonOrganizer` または `TenantAdmin` が Hackathon ごとのスカウト ON / OFF を設定できる。
- 送信権限を持つ `Sponsor` または `Judge` がアプリからスカウトを送信できる。

## 3. 基本構造
### 3.1. Tenant
Tenant は、主催団体や運営主体を表す管理単位である。  
`Sponsor` は Tenant に所属し、どの Hackathon を閲覧できるかは `TenantAdmin` が制御する。

### 3.2. Hackathon
Hackathon は、Tenant 配下にぶら下がるイベント単位である。  
`Judge` と `Hacker` は Hackathon 単位で所属する。  
`HackathonOrganizer` は特定 Hackathon の運営を担う。

### 3.3. ロールのスコープ

| ロール | スコープ |
| --- | --- |
| `PlatformAdmin` | Global |
| `TenantAdmin` | Tenant |
| `HackathonOrganizer` | Hackathon |
| `Sponsor` | Tenant |
| `Judge` | Hackathon |
| `Hacker` | Hackathon |

### 3.4. 複数ロール所持
- 同一ユーザーが複数ロールを持つことを許容する。
- 複数ロールはスコープ付きで保持する。
- 兼務は `TenantAdmin` と `HackathonOrganizer` のような管理ロール同士だけでなく、管理ロールと `Sponsor` / `Judge` のような送信ロールの組み合わせも許容する。
- `TenantAdmin` は自 Tenant 配下の Hackathon に対して、`HackathonOrganizer` 相当の上位権限を持つ。
- `HackathonOrganizer` は担当 Hackathon に限定して操作できる。
- ただし、管理ロールはスカウト送信権限を直接表さない。送信は `Sponsor` または送信権限を持つ `Judge` ロールでのみ行う。

### 3.5. 招待 URL と参加パスワード
参加導線は、対象範囲に応じた招待 URL と参加パスワードの組み合わせで提供する。

- Hackathon 参加用: `Hacker` / `Judge` 向け
- Tenant 参加用: `Sponsor` 向け

### 3.6. スカウトの対象範囲
スカウト送信は、送信者がアクセス可能な Hackathon に所属し、かつスカウト受信を許可している `Hacker` に対してのみ可能とする。

## 4. 主要ロールと責務

| ロール | 主な責務 |
| --- | --- |
| `PlatformAdmin` | Tenant 作成、Tenant の有効 / 無効、初期 `TenantAdmin` 付与 |
| `TenantAdmin` | Hackathon 作成、`HackathonOrganizer` 割り当て、Sponsor 招待、Sponsor 表示範囲設定、Tenant 内設定 |
| `HackathonOrganizer` | Hackathon 招待管理、Hackathon ごとのスカウト ON / OFF、Judge 送信可否設定 |
| `Sponsor` | 許可された Hackathon の閲覧、スカウト送信 |
| `Judge` | 担当 Hackathon の審査、必要時のスカウト送信 |
| `Hacker` | Hackathon 参加、スカウト受信可否設定、メールでのスカウト受領 |

補足:

- `PlatformAdmin` は最小グローバル権限であり、Hackathon の業務運営には直接関与しない。
- `TenantAdmin` は、Tenant 配下のすべての Hackathon に対する上位運営権限を持つ。
- `HackathonOrganizer` は、割り当てられた Hackathon のみ管理できる。
- `TenantAdmin` / `HackathonOrganizer` は管理主体であり、スカウト送信主体ではない。

## 5. 主要ユーザーフロー
### 5.1. PlatformAdmin の初期セットアップ
1. `PlatformAdmin` が Tenant を作成する。
2. 初期 `TenantAdmin` を割り当てる。

### 5.2. TenantAdmin の初期セットアップ
1. `TenantAdmin` が Tenant 配下に Hackathon を作成する。
2. 必要に応じて `HackathonOrganizer` を割り当てる。
3. `Sponsor` 用の Tenant 参加 URL と参加パスワードを発行する。
4. `Sponsor` ごとに閲覧可能な Hackathon 範囲を設定する。

### 5.3. HackathonOrganizer の初期セットアップ
1. `HackathonOrganizer` が `Hacker` / `Judge` 用の Hackathon 参加 URL と参加パスワードを発行する。
2. 招待の有効期限を `5 / 7 / 14 / 30 / 60 / 90日` から設定する。
3. Hackathon ごとのスカウト ON / OFF と Judge の送信可否を設定する。

### 5.4. Hacker / Judge の参加
1. 招待 URL にアクセスする。
2. 参加パスワードを入力する。
3. 新規登録またはログインを行う。
4. メール認証完了後、対象 Hackathon へ参加する。

### 5.5. Sponsor の参加
1. Tenant 参加用 URL にアクセスする。
2. 参加パスワードを入力する。
3. 新規登録またはログインを行う。
4. メール認証完了後、対象 Tenant の `Sponsor` として参加する。
5. `TenantAdmin` が許可した Hackathon だけを閲覧できる。

### 5.6. スカウト送信
1. `Sponsor` または送信権限を持つ `Judge` がログインする。
2. 閲覧可能な Hackathon を選択する。
3. 対象 `Hacker` を選択してスカウトを作成する。
4. 受信者が opt-in 中であり、Hackathon のスカウト機能が有効な場合のみ送信できる。
5. 送信後、`Hacker` の登録メールアドレスへ通知する。

## 6. 機能要件
### 6.1. Platform 管理
- `PlatformAdmin` は Tenant を作成できる。
- `PlatformAdmin` は Tenant の有効 / 無効を切り替えられる。
- `PlatformAdmin` は初期 `TenantAdmin` を設定できる。

### 6.2. Tenant 管理
- `TenantAdmin` は Tenant 配下に複数の Hackathon を作成できる。
- `TenantAdmin` は `HackathonOrganizer` を割り当てできる。
- `TenantAdmin` は `Sponsor` 用招待を発行できる。
- `TenantAdmin` は専用画面上で `Sponsor` ごとの閲覧可能 Hackathon 範囲を設定できる。

### 6.3. Hackathon 管理
- `HackathonOrganizer` は担当 Hackathon の招待を発行できる。
- `HackathonOrganizer` は担当 Hackathon のスカウト機能 ON / OFF を設定できる。
- `HackathonOrganizer` は担当 Hackathon の Judge 送信可否を設定できる。
- `TenantAdmin` は自 Tenant 配下のすべての Hackathon で同等の設定変更ができる。

### 6.4. 招待管理
- 招待 URL には参加パスワードを設定できる。
- 招待 URL には有効期限を設定できる。
- 有効期限は `5 / 7 / 14 / 30 / 60 / 90日` から選択できる。
- 招待 URL は再発行できる。
- 招待 URL は無効化できる。
- 招待は対象ロールを明示して発行する。

### 6.5. 参加と所属
- `Hacker` と `Judge` は Hackathon 単位で所属する。
- `Sponsor` は Tenant 単位で所属する。
- 参加完了にはメール認証を必須とする。
- 同一ユーザーは複数スコープで複数ロールを持てる。

### 6.6. Sponsor の閲覧範囲
- `TenantAdmin` は専用画面上で、`Sponsor` ごとに閲覧可能な Hackathon 範囲を設定できる。
- `Sponsor` は許可された Hackathon のみ閲覧できる。

### 6.7. スカウト送信
- `Sponsor` または送信権限を持つ `Judge` はアプリ内でスカウトを作成・送信できる。
- スカウト対象は、対象 Hackathon に所属し、受信を許可している `Hacker` に限定する。
- スカウト送信後、受信者へメール通知を送る。

### 6.8. スカウト受信設定
- `Hacker` は Hackathon 参加後に、スカウト受信の opt-in / opt-out を切り替えられる。
- opt-out 中の `Hacker` はスカウト対象に表示しない、またはアクセス不能として扱う。

## 7. アクセス制御要件
### 7.1. Rule 1: ロールはスコープ付きで付与する
- ロールは Global / Tenant / Hackathon のいずれかのスコープ付きで保持する。
- 同一ユーザーが複数ロールを持つことを許容する。

### 7.2. Rule 2: PlatformAdmin は最小グローバル権限
- `PlatformAdmin` は Tenant の作成と初期セットアップに限定してグローバル権限を持つ。
- `PlatformAdmin` が Tenant / Hackathon の業務運営データへ自動的にアクセスできる前提にはしない。

### 7.3. Rule 3: TenantAdmin は自 Tenant のみ管理可能
- `TenantAdmin` は自分が所属する Tenant 配下の Hackathon と Sponsor 設定のみ管理できる。
- 他 Tenant の設定にはアクセスできない。

### 7.4. Rule 4: HackathonOrganizer は担当 Hackathon のみ管理可能
- `HackathonOrganizer` は割り当てられた Hackathon のみ管理できる。
- 他 Hackathon の招待や設定にはアクセスできない。

### 7.5. Rule 5: Sponsor の閲覧範囲制御
- `Sponsor` は所属 Tenant 内であっても、`TenantAdmin` が許可した Hackathon 以外は閲覧できない。

### 7.6. Rule 6: スカウト許諾に基づく制御
- 送信者は opt-in 済みの `Hacker` にのみスカウトできる。
- opt-out 中の対象は、リソースが存在しないものとして扱う。

### 7.7. Rule 7: 招待の有効性
- 招待 URL は、発行対象の Tenant または Hackathon に対してのみ有効とする。
- 招待 URL と参加パスワードの両方を満たした場合のみ参加できる。
- 無効化済み、または有効期限切れの招待 URL は利用できない。

### 7.8. Rule 8: 審査情報の隔離
- `Sponsor` は `Judge` の採点データや審査メモにアクセスできない。
- 審査機能とスポンサー向け機能は明確に分離する。

### 7.9. Rule 9: スポンサー間の情報隔離
- `Sponsor` は他スポンサーの送信先、送信内容、検討状況を閲覧できない。

### 7.10. Rule 10: スカウト機能と Judge 送信権限
- Hackathon ごとのスカウト ON / OFF は `HackathonOrganizer` または `TenantAdmin` が設定できる。
- `TenantAdmin` / `HackathonOrganizer` は設定変更主体であり、管理ロール単体ではスカウト送信できない。
- `Judge` のスカウト送信可否は Hackathon 設定に従う。

## 8. 将来拡張
- 条件によるチーム検索
- AI によるチーム内容ヒアリング
- スカウトテンプレート
- 企業賞ワークフロー支援
- 監査ログの可視化

## 9. 懸念事項・未確定事項
- 招待 URL のデフォルト有効期限をどれにするか
- 参加パスワードの更新時に既参加者へどう影響させるか
- `TenantAdmin` と `HackathonOrganizer` を同時に持つユーザーの初期ホームをどうするか
- 同一ユーザーが複数 Tenant / 複数 Hackathon にまたがる場合のロール切替 UX
- 検索や AI 機能をどの時点で追加するか
