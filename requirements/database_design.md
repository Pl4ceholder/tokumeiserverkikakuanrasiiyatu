# SQLiteデータベース設計書（実装AI向け）

- 文書バージョン：1.0
- 状態：実装の基準案。実際のTShock/SSC連携とSQLiteプロバイダー確認後に、破壊的変更を避けて確定する。
- 参照：`requirements_summary.md`、`requirements.json`、`anonymous_terraria_server_spec.md`、`anonymous_terraria_api_validation_matrix.md`

## 1. 設計目標

このDBはプラグイン独自の状態だけを保存する。TShockが管理するアカウント、パスワード、SSCキャラクターデータの内部形式を独自に複製・置換しない。

満たすこと：

1. 日次ミッションはJSTの日付キーで一意に管理し、サーバー停止後も当日状態へ収束する。
2. JSONで選ばれたミッションの当日スナップショットと進捗を保存し、設定変更で進行中のミッションが変化しない。
3. 個人進捗、全体進捗、全体ミッションへの個人貢献を分離する。
4. 報酬の権利・付与状態を記録し、同じ報酬イベントを二重に作成しない。付与結果が不明な状態を自動再試行で二重配布しない。
5. ボスの解禁済み状態と管理者による手動介入履歴を保持する。スケジュール定義そのものはJSONを正とする。
6. 表彰10部門の集計値・発見履歴・ボス戦参加履歴・最終結果を保持する。公開ランキングは作成しない。
7. DBスキーマはバージョン管理し、起動時に安全にマイグレーションする。

## 2. 保存規約

- DBファイル名の初期案：`anonymous_server.sqlite`。配置場所はプラグイン設定で指定する。
- SQLiteの接続プロバイダーは、ターゲット環境で利用可能なものを最初の技術スパイクで確定する。TShock内部のDB APIや追加NuGetパッケージを、確認せずに利用しない。
- 接続ごとに `PRAGMA foreign_keys = ON` を有効にする。書き込み接続では適切な `busy_timeout` を設定する。
- WALモードは使用プロバイダー／サーバー環境で動作確認できた場合に採用する。バックアップはWALを考慮した整合性のある手段を使う。
- 全時刻はISO 8601 UTC文字列（例：`2026-10-16T15:00:00Z`）として保存する。ミッション日付キー `day_key` のみ `YYYY-MM-DD` のJST日付を使う。
- 内部IDはGUID文字列など不変の値。表示名を主キー・外部キーにしない。
- 金額は浮動小数点でなく、銅貨相当の整数（64-bit signed integer）で比較する。
- JSON列には秘密情報、パスワード、不要な個人情報を保存しない。
- 過去データは表彰・監査・報酬再照合に必要。削除／保持期間の方針が別途確定するまでは自動削除しない。

## 3. エンティティ関係の概要

```text
players ──< mission_player_progress >── mission_instances ── mission_global_progress
   │                    │                       │
   │                    └── mission_contribution_totals
   │
   ├──< reward_grants ── daily_bonus_claims
   ├──< award_metrics / award_discoveries / award_activity_days
   └──< boss_fight_participants >── boss_fights

award_seasons ──< award_targets
      │
      ├──< award_finalization_metrics
      └──< award_results ──< award_recipients

boss_unlock_state / boss_admin_actions / admin_audit_log / server_state
```

この図は概念図。実際のDDLと制約は以下を基準にする。

## 4. テーブル設計

### 4.1 `schema_migrations`

適用済みDDLのバージョン管理。起動時は未適用マイグレーションだけを順番に実行する。

| カラム | 型 | 制約・用途 |
|---|---|---|
| `version` | INTEGER | PRIMARY KEY |
| `name` | TEXT | NOT NULL |
| `checksum` | TEXT | NOT NULL。適用済みマイグレーションの改変検出 |
| `applied_at_utc` | TEXT | NOT NULL |

本番運用で既存マイグレーションを編集しない。修正は新しいバージョンとして追加する。

### 4.2 `players`

プラグイン内の不変な参加者IDと、検証後のTShockアカウントへの関連付け。自動ログイン方式は未検証なので、関連付けの実際の作成方法はAPI検証後に決める。

| カラム | 型 | 制約・用途 |
|---|---|---|
| `player_id` | TEXT | PRIMARY KEY。プラグイン内の不変ID |
| `tshock_account_id` | INTEGER | NULL可、UNIQUE。実際に安定したアカウントIDを取れる場合のみ保存 |
| `link_status` | TEXT | `pending` / `linked` / `needs_admin` / `inactive` |
| `last_display_name` | TEXT | NULL可。内部参照用。外部キーに使わない |
| `first_seen_at_utc` | TEXT | NOT NULL |
| `last_seen_at_utc` | TEXT | NOT NULL |
| `is_active` | INTEGER | NOT NULL、0/1 |

パスワードや認証秘密情報をこのテーブルに保存しない。`tshock_account_id`が取得できない場合に、接続UUIDや表示名だけを無検証で本人識別の唯一の根拠にしない。

### 4.3 `server_state`

サーバー全体で一つの状態を管理する小さなキー・バリューテーブル。

| カラム | 型 | 制約・用途 |
|---|---|---|
| `state_key` | TEXT | PRIMARY KEY |
| `value_json` | TEXT | NOT NULL。設定／状態値をJSONで保存 |
| `updated_at_utc` | TEXT | NOT NULL |

想定キー：`world_opened_at_utc`、`timezone`（初期値 `Asia/Tokyo`）、`last_processed_day_key`、`active_award_season_id`、`last_loaded_config_version`。共有報酬ティアは専用テーブルで管理する。

### 4.4 `mission_instances`

あるJST日付に選出されたミッションの実体。JSONの定義をそのまま参照し続けるのでなく、選出時のスナップショットを保存する。

| カラム | 型 | 制約・用途 |
|---|---|---|
| `mission_instance_id` | TEXT | PRIMARY KEY。GUID等 |
| `day_key` | TEXT | NOT NULL、JSTの `YYYY-MM-DD` |
| `scope` | TEXT | `global` / `personal` |
| `selection_slot` | INTEGER | NOT NULL。同日・同scope内のスロット番号 |
| `mission_id` | TEXT | NOT NULL。候補JSONの固定ID |
| `category` | TEXT | NOT NULL |
| `definition_snapshot_json` | TEXT | NOT NULL。条件・数量・報酬・進行条件等の選出時定義 |
| `target_value` | INTEGER | NULL可。数値目標がある場合 |
| `status` | TEXT | `active` / `completed` / `expired` / `disabled` |
| `selected_at_utc` | TEXT | NOT NULL |
| `starts_at_utc` | TEXT | NOT NULL |
| `ends_at_utc` | TEXT | NOT NULL |
| `completed_at_utc` | TEXT | NULL可 |

制約：`UNIQUE(day_key, scope, selection_slot)`。一日のミッション数は設定可能とし、`slot=0..N-1`を使う。同じ候補を同日に重複選出するかは選出ロジックで制御する。

### 4.5 `mission_global_progress`

全体ミッションの現在進捗を1実体につき1行で保持。

| カラム | 型 | 制約・用途 |
|---|---|---|
| `mission_instance_id` | TEXT | PRIMARY KEY、`mission_instances` FK |
| `progress_value` | INTEGER | NOT NULL DEFAULT 0 |
| `progress_state_json` | TEXT | NOT NULL DEFAULT `{}`。複合条件向け |
| `status` | TEXT | `active` / `completed` / `expired` |
| `updated_at_utc` | TEXT | NOT NULL |
| `completed_at_utc` | TEXT | NULL可 |

全体目標達成状態への遷移はトランザクション内で一度だけ行い、報酬付与レコード作成と同時に確定する。

### 4.6 `mission_player_progress`

個人ミッションの参加者別進捗。参加者が途中参加した場合は、その日のミッションへ参加した時点で行を作る。参加前の活動を遡ってカウントしない。

| カラム | 型 | 制約・用途 |
|---|---|---|
| `mission_instance_id` | TEXT | NOT NULL、`mission_instances` FK |
| `player_id` | TEXT | NOT NULL、`players` FK |
| `progress_value` | INTEGER | NOT NULL DEFAULT 0 |
| `progress_state_json` | TEXT | NOT NULL DEFAULT `{}` |
| `status` | TEXT | `active` / `completed` / `expired` |
| `started_at_utc` | TEXT | NOT NULL |
| `updated_at_utc` | TEXT | NOT NULL |
| `completed_at_utc` | TEXT | NULL可 |

PRIMARY KEY (`mission_instance_id`, `player_id`)。個人報酬の権利はこの行の状態だけから再発行せず、必ず`reward_grants`を参照する。

### 4.7 `mission_contribution_totals`

全体ミッションへの個人貢献を集計する。全体進捗の加算と同一トランザクションで更新する。これを物資支援賞・ミッション貢献賞の集計元に利用する場合、イベント種類別の内訳を `contribution_detail_json` に記録する。

| カラム | 型 | 制約・用途 |
|---|---|---|
| `mission_instance_id` | TEXT | NOT NULL、`mission_instances` FK |
| `player_id` | TEXT | NOT NULL、`players` FK |
| `contribution_value` | INTEGER | NOT NULL DEFAULT 0 |
| `contribution_detail_json` | TEXT | NOT NULL DEFAULT `{}` |
| `updated_at_utc` | TEXT | NOT NULL |

PRIMARY KEY (`mission_instance_id`, `player_id`)。これ単独でランキングを公開する機能は作らない。

### 4.8 `reward_grants`

個人ミッション、全体ミッション報酬、ログインボーナスなど、アイテム／効果付与の共通台帳。

| カラム | 型 | 制約・用途 |
|---|---|---|
| `grant_id` | TEXT | PRIMARY KEY |
| `idempotency_key` | TEXT | NOT NULL UNIQUE。同じ報酬権利を識別する決定的キー |
| `recipient_player_id` | TEXT | NOT NULL、`players` FK |
| `source_type` | TEXT | `personal_mission` / `global_mission` / `daily_bonus` / `other_configured` |
| `source_id` | TEXT | NOT NULL。ミッション実体ID等 |
| `day_key` | TEXT | NULL可。日次報酬ならJST日付 |
| `reward_payload_json` | TEXT | NOT NULL。アイテム・数量・効果のスナップショット |
| `status` | TEXT | `pending` / `processing` / `granted` / `needs_review` / `failed` |
| `attempt_count` | INTEGER | NOT NULL DEFAULT 0 |
| `created_at_utc` | TEXT | NOT NULL |
| `updated_at_utc` | TEXT | NOT NULL |
| `granted_at_utc` | TEXT | NULL可 |
| `last_error_code` | TEXT | NULL可 |
| `last_error_message` | TEXT | NULL可。秘密情報を含めない |

インデックス：(`recipient_player_id`, `status`)、(`status`, `created_at_utc`)、(`source_type`, `source_id`)。`idempotency_key`は例として `daily-bonus:{day_key}:{player_id}`、`mission-reward:{mission_instance_id}:{player_id}` のように作る。

**重要：** SQLiteのトランザクションとゲーム内アイテム／SSC保存を一つの原子的処理とはみなさない。付与結果が曖昧な強制終了ケースは `needs_review` にし、無条件の自動再配布を禁止する。

### 4.9 `daily_bonus_claims`

日次ボーナス受取の一意性を管理する。

| カラム | 型 | 制約・用途 |
|---|---|---|
| `player_id` | TEXT | NOT NULL、`players` FK |
| `day_key` | TEXT | NOT NULL。JST日付 |
| `grant_id` | TEXT | NOT NULL UNIQUE、`reward_grants` FK |
| `tier_at_claim` | INTEGER | NOT NULL。受取開始時のティアを固定 |
| `claimed_at_utc` | TEXT | NOT NULL。受取処理の開始時刻 |

PRIMARY KEY (`player_id`, `day_key`)。最初の操作時にclaimとgrantを同一トランザクションで作成する。2回目以降は新しいgrantを作らず、既存grantの状態に応じて「受取済み／受取待ち」を処理する。

### 4.10 `shared_reward_state`

累積全体実績で育つ共有報酬ティア。日次リセットでは低下させない。

| カラム | 型 | 制約・用途 |
|---|---|---|
| `singleton_id` | INTEGER | PRIMARY KEY、常に1 |
| `current_tier` | INTEGER | NOT NULL |
| `cumulative_global_completions` | INTEGER | NOT NULL DEFAULT 0 |
| `updated_at_utc` | TEXT | NOT NULL |

行がなければ初期ティアを作る。ティア上昇判定はトランザクション内で行い、設定変更で既存ティアを自動降格させない。

### 4.11 `boss_unlock_state`

ボスごとの解禁済み状態。予定表は `boss_schedule.json` が正、ここは実際の状態と履歴のための記録。

| カラム | 型 | 制約・用途 |
|---|---|---|
| `boss_key` | TEXT | PRIMARY KEY。内部で固定のボス識別子 |
| `scheduled_at_utc` | TEXT | NULL可。現在設定から計算した予定時刻 |
| `is_unlocked` | INTEGER | NOT NULL、0/1 |
| `unlocked_at_utc` | TEXT | NULL可 |
| `unlock_source` | TEXT | NULL可。`schedule` / `manual` |
| `last_checked_at_utc` | TEXT | NOT NULL |

起動時に時刻を再計算する。既に解禁済みのボスは、設定変更で再ロックしない。Terrariaの標準進行条件はこの表の解禁状態と別に検査する。

### 4.12 `boss_admin_actions`

管理者による解禁などの手動介入履歴。

`action_id TEXT PRIMARY KEY`、`boss_key TEXT NOT NULL`、`action TEXT NOT NULL`、`actor_player_id TEXT NULL`、`occurred_at_utc TEXT NOT NULL`、`reason TEXT NULL`、`details_json TEXT NOT NULL DEFAULT '{}'`。

### 4.13 `award_seasons`

表彰期間の管理。期間と基準は開始前に固定し、運用途中の無断変更を避ける。

`season_id TEXT PRIMARY KEY`、`name TEXT NOT NULL`、`starts_at_utc TEXT NOT NULL`、`ends_at_utc TEXT NOT NULL`、`status TEXT NOT NULL`（`planned` / `active` / `closed` / `finalized`）、`rules_snapshot_json TEXT NOT NULL`、`created_at_utc TEXT NOT NULL`、`finalized_at_utc TEXT NULL`。

### 4.14 `award_targets`

皆勤賞の対象ボス。設定ファイルの値を開催期間用スナップショットとして保存する。

PRIMARY KEY (`season_id`, `boss_key`)、`season_id`は`award_seasons`へのFK。表彰開始後、対象ボスの変更は既存シーズンへ暗黙適用しない。

### 4.15 `award_metrics`

表彰用の単純累積カウンター。`metric_key`はコード／設定で許可した固定キーだけを受け付ける。

| カラム | 型 | 用途 |
|---|---|---|
| `season_id` | TEXT | NOT NULL、FK |
| `player_id` | TEXT | NOT NULL、FK |
| `metric_key` | TEXT | NOT NULL。例：`mined_blocks`、`placed_blocks`、`flowering_herbs`、`fish_catches`、`delivered_units` |
| `metric_value` | INTEGER | NOT NULL DEFAULT 0。負数は不可 |
| `details_json` | TEXT | NOT NULL DEFAULT `{}`。必要な内訳のみ |
| `updated_at_utc` | TEXT | NOT NULL |

PRIMARY KEY (`season_id`, `player_id`, `metric_key`)。公開用のランキングAPIを作らない。イベント検出が曖昧な指標は加算せず、実装ブロッカーとして扱う。

### 4.16 `award_discoveries`

開拓功労賞用の初回発見記録。同一プレイヤー・同一シーズン・同一の正規化探索キーは一度だけ記録。

PRIMARY KEY (`season_id`, `player_id`, `discovery_key`)、`discovered_at_utc TEXT NOT NULL`、`details_json TEXT NOT NULL DEFAULT '{}'`。キーの粒度（バイオーム、探索地点等）はシーズン開始前に固定する。

### 4.17 `award_activity_days`

継続冒険賞用。実際の活動が閾値を超えた日だけ1行記録する。単にログインしているだけでは記録しない。

PRIMARY KEY (`season_id`, `player_id`, `day_key`)、`activity_score INTEGER NOT NULL`、`first_activity_at_utc TEXT NOT NULL`、`last_activity_at_utc TEXT NOT NULL`。

### 4.18 `boss_fights` と `boss_fight_participants`

`boss_fights`：`fight_id TEXT PRIMARY KEY`、`boss_key TEXT NOT NULL`、`started_at_utc TEXT NOT NULL`、`ended_at_utc TEXT NULL`、`status TEXT NOT NULL`。派生部位／複数パーツは、検証済みの識別ルールで同一戦闘へまとめる。

`boss_fight_participants`：PRIMARY KEY (`fight_id`, `player_id`)、FKは戦闘とプレイヤー。`qualifying_reason TEXT NOT NULL`、`first_qualifying_at_utc TEXT NOT NULL`、`last_qualifying_at_utc TEXT NOT NULL`、`qualifying_event_count INTEGER NOT NULL DEFAULT 1`。

召喚者であることや最後の一撃だけを参加基準にしない。遠距離・投射物・召喚攻撃も含む参加判定が技術検証で成立すること。AFK滞在のみは参加とみなさない。

### 4.19 `award_finalization_metrics`

`/award preview`時の候補集計、または`/award finalize`直前の確定スナップショットを内部用に保存し、結果を再現可能にする。値は公開しない。

PRIMARY KEY (`season_id`, `award_key`, `player_id`)、`metric_value INTEGER NOT NULL`、`snapshot_at_utc TEXT NOT NULL`、`evidence_json TEXT NOT NULL DEFAULT '{}'`。富豪は確定時点の個人資産をここへ保存する。オフラインプレイヤーのSSCデータから資産を正確に取得できるかは技術検証が必要。

### 4.20 `award_results` と `award_recipients`

`award_results`：PRIMARY KEY (`season_id`, `award_key`)、`outcome TEXT NOT NULL`（`awarded` / `no_recipient`）、`finalized_at_utc TEXT NOT NULL`、`criteria_snapshot_json TEXT NOT NULL`。

`award_recipients`：PRIMARY KEY (`season_id`, `award_key`, `player_id`)、`display_name_snapshot TEXT NOT NULL`、`metric_value_private INTEGER NULL`、`details_private_json TEXT NOT NULL DEFAULT '{}'`。

同率は複数行を受賞者に保存する。`metric_value_private`や`details_private_json`を一般参加者向けに表示しない。公開発表は賞名・受賞者表示名だけ。

### 4.21 `admin_audit_log`

管理者操作と重大状態変更を追跡する。

`audit_id TEXT PRIMARY KEY`、`actor_player_id TEXT NULL`、`actor_type TEXT NOT NULL`（`admin_player` / `server_console` / `system`）、`action TEXT NOT NULL`、`target_type TEXT NULL`、`target_id TEXT NULL`、`occurred_at_utc TEXT NOT NULL`、`details_json TEXT NOT NULL DEFAULT '{}'`。

パスワード、認証秘密情報、不要な個人情報は記録しない。

## 5. 初期スキーマDDL

以下は初期マイグレーションの基準DDL。実装前に対象SQLiteプロバイダーで実行し、全FK・CHECK・UNIQUE制約を確認すること。カラムや制約を変える場合は、この資料とマイグレーションを同じ変更で更新する。

```sql
PRAGMA foreign_keys = ON;

CREATE TABLE IF NOT EXISTS schema_migrations (
  version INTEGER PRIMARY KEY,
  name TEXT NOT NULL,
  checksum TEXT NOT NULL,
  applied_at_utc TEXT NOT NULL
);

CREATE TABLE IF NOT EXISTS players (
  player_id TEXT PRIMARY KEY,
  tshock_account_id INTEGER UNIQUE,
  link_status TEXT NOT NULL CHECK (link_status IN ('pending','linked','needs_admin','inactive')),
  last_display_name TEXT,
  first_seen_at_utc TEXT NOT NULL,
  last_seen_at_utc TEXT NOT NULL,
  is_active INTEGER NOT NULL DEFAULT 1 CHECK (is_active IN (0,1))
);

CREATE TABLE IF NOT EXISTS server_state (
  state_key TEXT PRIMARY KEY,
  value_json TEXT NOT NULL,
  updated_at_utc TEXT NOT NULL
);

CREATE TABLE IF NOT EXISTS mission_instances (
  mission_instance_id TEXT PRIMARY KEY,
  day_key TEXT NOT NULL,
  scope TEXT NOT NULL CHECK (scope IN ('global','personal')),
  selection_slot INTEGER NOT NULL,
  mission_id TEXT NOT NULL,
  category TEXT NOT NULL,
  definition_snapshot_json TEXT NOT NULL,
  target_value INTEGER,
  status TEXT NOT NULL CHECK (status IN ('active','completed','expired','disabled')),
  selected_at_utc TEXT NOT NULL,
  starts_at_utc TEXT NOT NULL,
  ends_at_utc TEXT NOT NULL,
  completed_at_utc TEXT,
  UNIQUE(day_key, scope, selection_slot)
);

CREATE TABLE IF NOT EXISTS mission_global_progress (
  mission_instance_id TEXT PRIMARY KEY REFERENCES mission_instances(mission_instance_id),
  progress_value INTEGER NOT NULL DEFAULT 0 CHECK (progress_value >= 0),
  progress_state_json TEXT NOT NULL DEFAULT '{}',
  status TEXT NOT NULL CHECK (status IN ('active','completed','expired')),
  updated_at_utc TEXT NOT NULL,
  completed_at_utc TEXT
);

CREATE TABLE IF NOT EXISTS mission_player_progress (
  mission_instance_id TEXT NOT NULL REFERENCES mission_instances(mission_instance_id),
  player_id TEXT NOT NULL REFERENCES players(player_id),
  progress_value INTEGER NOT NULL DEFAULT 0 CHECK (progress_value >= 0),
  progress_state_json TEXT NOT NULL DEFAULT '{}',
  status TEXT NOT NULL CHECK (status IN ('active','completed','expired')),
  started_at_utc TEXT NOT NULL,
  updated_at_utc TEXT NOT NULL,
  completed_at_utc TEXT,
  PRIMARY KEY (mission_instance_id, player_id)
);

CREATE TABLE IF NOT EXISTS mission_contribution_totals (
  mission_instance_id TEXT NOT NULL REFERENCES mission_instances(mission_instance_id),
  player_id TEXT NOT NULL REFERENCES players(player_id),
  contribution_value INTEGER NOT NULL DEFAULT 0 CHECK (contribution_value >= 0),
  contribution_detail_json TEXT NOT NULL DEFAULT '{}',
  updated_at_utc TEXT NOT NULL,
  PRIMARY KEY (mission_instance_id, player_id)
);

CREATE TABLE IF NOT EXISTS reward_grants (
  grant_id TEXT PRIMARY KEY,
  idempotency_key TEXT NOT NULL UNIQUE,
  recipient_player_id TEXT NOT NULL REFERENCES players(player_id),
  source_type TEXT NOT NULL,
  source_id TEXT NOT NULL,
  day_key TEXT,
  reward_payload_json TEXT NOT NULL,
  status TEXT NOT NULL CHECK (status IN ('pending','processing','granted','needs_review','failed')),
  attempt_count INTEGER NOT NULL DEFAULT 0 CHECK (attempt_count >= 0),
  created_at_utc TEXT NOT NULL,
  updated_at_utc TEXT NOT NULL,
  granted_at_utc TEXT,
  last_error_code TEXT,
  last_error_message TEXT
);

CREATE TABLE IF NOT EXISTS daily_bonus_claims (
  player_id TEXT NOT NULL REFERENCES players(player_id),
  day_key TEXT NOT NULL,
  grant_id TEXT NOT NULL UNIQUE REFERENCES reward_grants(grant_id),
  tier_at_claim INTEGER NOT NULL CHECK (tier_at_claim >= 1),
  claimed_at_utc TEXT NOT NULL,
  PRIMARY KEY (player_id, day_key)
);

CREATE TABLE IF NOT EXISTS shared_reward_state (
  singleton_id INTEGER PRIMARY KEY CHECK (singleton_id = 1),
  current_tier INTEGER NOT NULL CHECK (current_tier >= 1),
  cumulative_global_completions INTEGER NOT NULL DEFAULT 0 CHECK (cumulative_global_completions >= 0),
  updated_at_utc TEXT NOT NULL
);

CREATE TABLE IF NOT EXISTS boss_unlock_state (
  boss_key TEXT PRIMARY KEY,
  scheduled_at_utc TEXT,
  is_unlocked INTEGER NOT NULL DEFAULT 0 CHECK (is_unlocked IN (0,1)),
  unlocked_at_utc TEXT,
  unlock_source TEXT CHECK (unlock_source IS NULL OR unlock_source IN ('schedule','manual')),
  last_checked_at_utc TEXT NOT NULL
);

CREATE TABLE IF NOT EXISTS boss_admin_actions (
  action_id TEXT PRIMARY KEY,
  boss_key TEXT NOT NULL,
  action TEXT NOT NULL,
  actor_player_id TEXT REFERENCES players(player_id),
  occurred_at_utc TEXT NOT NULL,
  reason TEXT,
  details_json TEXT NOT NULL DEFAULT '{}'
);

CREATE TABLE IF NOT EXISTS award_seasons (
  season_id TEXT PRIMARY KEY,
  name TEXT NOT NULL,
  starts_at_utc TEXT NOT NULL,
  ends_at_utc TEXT NOT NULL,
  status TEXT NOT NULL CHECK (status IN ('planned','active','closed','finalized')),
  rules_snapshot_json TEXT NOT NULL,
  created_at_utc TEXT NOT NULL,
  finalized_at_utc TEXT
);

CREATE TABLE IF NOT EXISTS award_targets (
  season_id TEXT NOT NULL REFERENCES award_seasons(season_id),
  boss_key TEXT NOT NULL,
  PRIMARY KEY (season_id, boss_key)
);

CREATE TABLE IF NOT EXISTS award_metrics (
  season_id TEXT NOT NULL REFERENCES award_seasons(season_id),
  player_id TEXT NOT NULL REFERENCES players(player_id),
  metric_key TEXT NOT NULL,
  metric_value INTEGER NOT NULL DEFAULT 0 CHECK (metric_value >= 0),
  details_json TEXT NOT NULL DEFAULT '{}',
  updated_at_utc TEXT NOT NULL,
  PRIMARY KEY (season_id, player_id, metric_key)
);

CREATE TABLE IF NOT EXISTS award_discoveries (
  season_id TEXT NOT NULL REFERENCES award_seasons(season_id),
  player_id TEXT NOT NULL REFERENCES players(player_id),
  discovery_key TEXT NOT NULL,
  discovered_at_utc TEXT NOT NULL,
  details_json TEXT NOT NULL DEFAULT '{}',
  PRIMARY KEY (season_id, player_id, discovery_key)
);

CREATE TABLE IF NOT EXISTS award_activity_days (
  season_id TEXT NOT NULL REFERENCES award_seasons(season_id),
  player_id TEXT NOT NULL REFERENCES players(player_id),
  day_key TEXT NOT NULL,
  activity_score INTEGER NOT NULL CHECK (activity_score >= 0),
  first_activity_at_utc TEXT NOT NULL,
  last_activity_at_utc TEXT NOT NULL,
  PRIMARY KEY (season_id, player_id, day_key)
);

CREATE TABLE IF NOT EXISTS boss_fights (
  fight_id TEXT PRIMARY KEY,
  boss_key TEXT NOT NULL,
  started_at_utc TEXT NOT NULL,
  ended_at_utc TEXT,
  status TEXT NOT NULL CHECK (status IN ('active','completed','aborted','unknown'))
);

CREATE TABLE IF NOT EXISTS boss_fight_participants (
  fight_id TEXT NOT NULL REFERENCES boss_fights(fight_id),
  player_id TEXT NOT NULL REFERENCES players(player_id),
  qualifying_reason TEXT NOT NULL,
  first_qualifying_at_utc TEXT NOT NULL,
  last_qualifying_at_utc TEXT NOT NULL,
  qualifying_event_count INTEGER NOT NULL DEFAULT 1 CHECK (qualifying_event_count >= 1),
  PRIMARY KEY (fight_id, player_id)
);

CREATE TABLE IF NOT EXISTS award_finalization_metrics (
  season_id TEXT NOT NULL REFERENCES award_seasons(season_id),
  award_key TEXT NOT NULL,
  player_id TEXT NOT NULL REFERENCES players(player_id),
  metric_value INTEGER NOT NULL,
  snapshot_at_utc TEXT NOT NULL,
  evidence_json TEXT NOT NULL DEFAULT '{}',
  PRIMARY KEY (season_id, award_key, player_id)
);

CREATE TABLE IF NOT EXISTS award_results (
  season_id TEXT NOT NULL REFERENCES award_seasons(season_id),
  award_key TEXT NOT NULL,
  outcome TEXT NOT NULL CHECK (outcome IN ('awarded','no_recipient')),
  finalized_at_utc TEXT NOT NULL,
  criteria_snapshot_json TEXT NOT NULL,
  PRIMARY KEY (season_id, award_key)
);

CREATE TABLE IF NOT EXISTS award_recipients (
  season_id TEXT NOT NULL,
  award_key TEXT NOT NULL,
  player_id TEXT NOT NULL REFERENCES players(player_id),
  display_name_snapshot TEXT NOT NULL,
  metric_value_private INTEGER,
  details_private_json TEXT NOT NULL DEFAULT '{}',
  PRIMARY KEY (season_id, award_key, player_id),
  FOREIGN KEY (season_id, award_key) REFERENCES award_results(season_id, award_key)
);

CREATE TABLE IF NOT EXISTS admin_audit_log (
  audit_id TEXT PRIMARY KEY,
  actor_player_id TEXT REFERENCES players(player_id),
  actor_type TEXT NOT NULL CHECK (actor_type IN ('admin_player','server_console','system')),
  action TEXT NOT NULL,
  target_type TEXT,
  target_id TEXT,
  occurred_at_utc TEXT NOT NULL,
  details_json TEXT NOT NULL DEFAULT '{}'
);

CREATE INDEX IF NOT EXISTS ix_mission_instances_day ON mission_instances(day_key, scope, status);
CREATE INDEX IF NOT EXISTS ix_player_mission_status ON mission_player_progress(player_id, status);
CREATE INDEX IF NOT EXISTS ix_contribution_player ON mission_contribution_totals(player_id, mission_instance_id);
CREATE INDEX IF NOT EXISTS ix_reward_recipient_status ON reward_grants(recipient_player_id, status);
CREATE INDEX IF NOT EXISTS ix_reward_status_created ON reward_grants(status, created_at_utc);
CREATE INDEX IF NOT EXISTS ix_award_metric_lookup ON award_metrics(season_id, metric_key, metric_value);
CREATE INDEX IF NOT EXISTS ix_boss_fight_lookup ON boss_fights(boss_key, started_at_utc);
CREATE INDEX IF NOT EXISTS ix_admin_audit_lookup ON admin_audit_log(occurred_at_utc, action);
```

このSQLは初期スキーマ案として扱い、対象SQLiteプロバイダーでテストする。必要なマイグレーションが確定したら、各バージョンのDDLをリポジトリに固定し、実装時にこの資料と矛盾させない。

## 6. トランザクションと冪等性ルール

### 日次更新

1. 現在のJST `day_key` を計算する。
2. `server_state.last_processed_day_key` と比較する。
3. 過去日を再実行せず、現行日の選出を1回だけ作成する。
4. `mission_instances`と初期進捗をトランザクションで保存し、日次キーを同時に更新する。
5. 既に同じ日の同じslotがあれば再生成しない。

### ミッション達成・全体報酬

1. 成功したゲームイベントを検証する。
2. 進捗と個人貢献を同一トランザクションで更新する。
3. 達成状態への遷移を条件付き更新で一度だけ実行する。
4. 達成した時点以前に登録された参加者を対象に、決定的な`idempotency_key`で個別grantを作る。
5. grant作成と達成済み状態を同一DBトランザクションで保存する。

### アイテム付与

ゲーム内アイテム／SSC保存とSQLiteは単一の原子トランザクションではない。状態を`pending → processing → granted`で管理するが、起動時に`processing`だったものを無条件で再配布しない。付与結果を信頼できる方法で照合できなければ`needs_review`にする。

### ログインボーナス

- 一意キーは `(player_id, day_key)`。
- claim行とreward grantを同一DBトランザクションで作る。
- インベントリ満杯はclaim取消しではなく、同じgrantを受取待ちで保持する。
- 既にclaim行があるときは新しいgrantを作成しない。

### 表彰確定

- `award_seasons.status`が`finalized`なら通常の`finalize`を拒否する。
- まず候補と証拠をスナップショットし、同率者を含む受賞結果を1トランザクションで保存する。
- 公開発表はDB確定後に行う。発表送信だけが失敗した場合、同一結果の二重発表を避けるため送信状態をログで確認できるようにする。

## 7. 最低限のインデックス

- `mission_instances(day_key, scope, status)`
- `mission_player_progress(player_id, status)`
- `mission_contribution_totals(player_id, mission_instance_id)`
- `reward_grants(recipient_player_id, status)`
- `reward_grants(status, created_at_utc)`
- `award_metrics(season_id, metric_key, metric_value)`（実クエリ計画で有効性を確認して追加）
- `boss_fights(boss_key, started_at_utc)`
- `admin_audit_log(occurred_at_utc, action)`

インデックスは実際の問い合わせパターンと`EXPLAIN QUERY PLAN`を確認し、過剰に追加しない。

## 8. マイグレーション・復旧

- DBが存在しない場合だけ初期作成する。既存ファイルを初期化・上書きしない。
- マイグレーション前にバックアップを取る。マイグレーションは可能な範囲でトランザクション内で実行する。
- マイグレーション途中で失敗したら、サーバーを機能制限状態で起動するか起動を中止し、報酬配布を続行しない。
- 起動時に`PRAGMA foreign_key_check`と`PRAGMA integrity_check`を適切な頻度で実行する。高頻度の重い検査は毎接続で行わない。
- `processing`状態の付与は自動で再配布せず、検証後に`pending`へ戻せるものと`needs_review`にするものを区別する。
- バックアップから復旧する際は、ワールド・TShock/SSCデータ・プラグインDB・JSONの整合する世代を選ぶ。

## 9. 未確定事項・検証ゲート

- 利用するSQLiteプロバイダー／パッケージと正確な依存関係。
- TShockアカウントIDを安定して取得し、公開表示名と切り離した自動ログインをどう実現するか。
- SSCへのアイテム付与結果とプラグインDB台帳を照合できる範囲。
- 開花ハーブ・通常釣果・ボス参加等の計測結果をどのフックで確実に得られるか。
- オフラインプレイヤーのSSC保管コインを読み、富豪を公平に判定できるか。

これらが未検証でもDB構造だけは作れるが、実測できない指標を確定済みとして記録したり、報酬処理が原子的だと仮定したりしてはならない。
