# テーブル定義書

> システム: crowdodge（チーム「なせばなる」）混雑回避パーソナライズ型予定管理アプリ
> DB: PostgreSQL + PostGIS

## 設計前提

- カレンダーの Source of Truth は Google カレンダー。サーバは個別の予定（具体的な開始/終了時刻を持つ単発インスタンス）のみを近未来のローリング窓に投影として保持する。繰り返しルールはサーバに持たない。
- 命名規約: テーブル・列は snake_case。主キーは `<単数テーブル名>_uuid`（uuid型）。外部キー列名は親PKと一致。
- すべてのテーブルは `created_at` / `updated_at`（timestamp, NOT NULL）を持つ（明記しない場合も同様）。

## テーブル一覧

| No. | テーブル名 | 概要 |
|---|---|---|
| 1 | users | ユーザーのアカウント基本情報 |
| 2 | user_settings | ユーザーのリマインドタイミング等の設定 |
| 3 | user_calendars | ユーザーが混雑回避の対象に選択した Google カレンダー |
| 4 | event_calendar_syncs | Google カレンダー取り込みの同期・watch 状態（user_calendars と 1:1） |
| 5 | user_subscriptions | サブスクリプション登録状況（RevenueCat 参照） |
| 6 | user_devices | 通知対象デバイスの識別子（FCM）管理 |
| 7 | events | 個別の予定（Google カレンダーの投影） |
| 8 | event_destinations | 予定の目的地・ルート情報（シリーズ単位で共有） |
| 9 | event_destination_links | 予定と目的地グループのメンバーシップ |
| 10 | event_congestion_predictions | 予定の混雑予測 |
| 11 | notification_schedules | 通知タイミングの管理（サーバ常時監視） |

---

## 1. users — アカウント基本情報

| 列(物理) | データ型 | PK | NOT NULL | 参照 | 備考 |
|---|---|---|---|---|---|
| user_uuid | uuid | ● | ● | | ユーザーID |
| google_id | text | | ● | | Google ID |
| email | text | | ● | | メールアドレス |
| created_at | timestamp | | ● | | |
| updated_at | timestamp | | ● | | |

## 2. user_settings — ユーザー設定

| 列(物理) | データ型 | PK | NOT NULL | 参照 | 備考 |
|---|---|---|---|---|---|
| user_uuid | uuid | ● | ● | users.user_uuid | ユーザーID |
| home | point | | ● | | 自宅座標 |
| remind_timing | interval | | ● | | リマインドタイミング既定値 |
| created_at | timestamp | | ● | | |
| updated_at | timestamp | | ● | | |

## 3. user_calendars — 選択カレンダー

ユーザーが混雑回避の対象に選択した Google カレンダー。同期・watch 状態は `event_calendar_syncs` が保持する。

| 列(物理) | データ型 | PK | NOT NULL | 一意 | 参照 | 備考 |
|---|---|---|---|---|---|---|
| user_calendar_uuid | uuid | ● | ● | | | ユーザーカレンダーID |
| user_uuid | uuid | | ● | | users.user_uuid | ユーザーID |
| google_calendar_id | text | | ● | | | Google カレンダーID |
| created_at | timestamp | | ● | | | |
| updated_at | timestamp | | ● | | | |

- `UNIQUE(user_uuid, google_calendar_id)`: 同一ユーザーが同一カレンダーを重複登録しない。1ユーザー複数カレンダー・共有カレンダーの複数ユーザー登録はいずれも許容する。

## 4. event_calendar_syncs — カレンダー同期・watch 状態

Google カレンダーを `events` 投影へ取り込むための同期・watch 状態。`user_calendars` と 1:1。取り込み手順は [google-calendar-sync.md](google-calendar-sync.md) §5。

| 列(物理) | データ型 | PK | NOT NULL | 一意 | 参照 | 備考 |
|---|---|---|---|---|---|---|
| user_calendar_uuid | uuid | ● | ● | | user_calendars.user_calendar_uuid | 由来カレンダー（下流→上流参照） |
| materialized_until | timestamptz | | | | | ローリング窓で実体化済みの将来端（不在削除の境界） |
| watch_channel_id | text | | | ● | | Google Push チャンネルID（webhook ルーティング） |
| watch_resource_id | text | | | | | watch 対象リソースID（channels.stop に使用） |
| watch_channel_token | text | | | | | webhook 検証トークン（X-Goog-Channel-Token の照合用） |
| watch_expiration | timestamptz | | | | | watch チャンネル失効時刻（renew sweep 用・index） |
| created_at | timestamp | | ● | | | |
| updated_at | timestamp | | ● | | | |

- `watch_expiration` に index（renew 抽出用）。

## 5. user_subscriptions — サブスクリプション

| 列(物理) | データ型 | PK | NOT NULL | 参照 | 備考 |
|---|---|---|---|---|---|
| user_subscription_uuid | uuid | ● | ● | | ユーザー課金ID |
| user_uuid | uuid | | | users.user_uuid | ユーザーID |
| plan_name | text | | | | プラン名（Free / Premium） |
| status | text | | | | ステータス |
| expires_at | timestamp | | | | 有効期限 |
| rc_original_transaction_id | text | | ● | | サブスク管理サービス（RevenueCat）取引ID |
| created_at | timestamp | | ● | | |
| updated_at | timestamp | | ● | | |

## 6. user_devices — 通知デバイス

| 列(物理) | データ型 | PK | NOT NULL | 参照 | 備考 |
|---|---|---|---|---|---|
| device_uuid | uuid | ● | ● | | デバイスID |
| user_uuid | uuid | | ● | users.user_uuid | ユーザーID |
| fcm_token | text | | ● | | デバイストークン（FCM） |
| created_at | timestamp | | ● | | |
| updated_at | timestamp | | ● | | |

## 7. events — 個別の予定（Google カレンダーの投影）

自社ドメイン（混雑予測・目的地推定・リマインド・通知本文）が消費するフィールドのみを保持する。表示専用フィールド（色・参加者・公開範囲等）は保持しない。

| 列(物理) | データ型 | PK | NOT NULL | 一意 | 参照 | 備考 |
|---|---|---|---|---|---|---|
| event_uuid | uuid | ● | ● | | | イベントID |
| user_calendar_uuid | uuid | | ● | | user_calendars.user_calendar_uuid | 由来カレンダー |
| google_event_id | text | | ● | ● | | Google イベント（インスタンス）ID。突合キー |
| recurring_event_id | text | | | | | Google シリーズ（マスタ）ID。単発は null |
| original_start | timestamp | | | | | RECURRENCE-ID（編集スコープ判定の分割点） |
| etag | text | | | | | Google etag（競合・ループ防止判定用） |
| title | text | | ● | | | タイトル |
| description | text | | | | | 概要 |
| location | text | | | | | 場所 |
| start_time | timestamp | | ● | | | 開始時刻 |
| end_time | timestamp | | ● | | | 終了時刻 |
| is_all_day | boolean | | ● | | | 終日予定フラグ |
| remind_timing | interval | | | | | リマインド間隔。null は user_settings 参照 |
| created_at | timestamp | | ● | | | |
| updated_at | timestamp | | ● | | | |

## 8. event_destinations — 目的地・ルート情報

目的地は場所に紐づき日付に依存しない。同一シリーズの全発生回で1グループを共有する（`recurring_event_id` を相関ヒントとして重複を防ぐ）。

| 列(物理) | データ型 | PK | NOT NULL | 一意 | 備考 |
|---|---|---|---|---|---|
| event_destination_uuid | uuid | ● | ● | | 目的地グループID |
| recurring_event_id | text | | | ● | シリーズ相関ヒント。単発は null |
| destination | text | | ● | | 目的地 |
| destination_point | point | | ● | | 目的地座標 |
| route_duration | interval | | ● | | 所要時間 |
| route_information | json | | ● | | ルート情報（LLM入力用） |
| created_at | timestamp | | ● | | |
| updated_at | timestamp | | ● | | |

## 9. event_destination_links — 予定↔目的地グループ

| 列(物理) | データ型 | PK | NOT NULL | 参照 | 備考 |
|---|---|---|---|---|---|
| event_uuid | uuid | ● | ● | events.event_uuid | 予定 |
| event_destination_uuid | uuid | | ● | event_destinations.event_destination_uuid | 所属グループ |
| created_at | timestamp | | ● | | |

## 10. event_congestion_predictions — 混雑予測

1予定につき1行（再予測は上書き）。

| 列(物理) | データ型 | PK | NOT NULL | 一意 | 参照 | 備考 |
|---|---|---|---|---|---|---|
| event_congestion_prediction_uuid | uuid | ● | ● | | | イベント混雑ID |
| event_uuid | uuid | | ● | ● | events.event_uuid | 予定 |
| congestion_start_time | timestamp | | ● | | | 混雑開始日時 |
| congestion_end_time | timestamp | | ● | | | 混雑終了日時 |
| description | text | | ● | | | 概要 |
| created_at | timestamp | | ● | | | |
| updated_at | timestamp | | ● | | | |

## 11. notification_schedules — 通知スケジュール

通知ジョブのキュー。サーバが常時ポーリングし、`notificate_time` 到来分を発火する。

| 列(物理) | データ型 | PK | NOT NULL | 参照 | 備考 |
|---|---|---|---|---|---|
| notification_schedule_uuid | uuid | ● | ● | | 通知スケジュールID |
| user_uuid | uuid | | ● | users.user_uuid | ユーザーID |
| event_uuid | uuid | | ● | events.event_uuid | 予定 |
| notificate_time | timestamp | | ● | | 通知時刻 |
| kind | text | | ● | | 通知種別: Reminder（必須=予定前リマインド） / CongestionAlert（任意=混雑通知） |
| status | text | | ● | | ステータス: pending / processing / completed / failed / cancelled |
| created_at | timestamp | | ● | | |
| updated_at | timestamp | | ● | | |
