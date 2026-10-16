# AI向け要件パッケージ

これは公開紹介ページではなく、設計・実装AI向けの内部要件です。静的公開する場合は `docs/` だけをGitHub Pagesの公開元にしてください。

## 読む順番
1. `CODING_AI_HANDOFF.md` — 作業ルール、守るべき制約、初回タスク。
2. `requirements_summary.md` — 決定事項の短い要約。
3. `requirements.json` — 要件ID／優先度／状態の機械可読インデックス。
4. `anonymous_terraria_server_spec.md` — 統合仕様書・詳細要件・受け入れ基準。
5. `database_design.md` — SQLiteテーブル、制約、トランザクション、冪等性方針。
6. `anonymous_terraria_api_validation_matrix.md` — 公開ソース事前調査と実機検証が必要なブロッカー。
7. `implementation_plan.md` — 段階実装、依存関係、各フェーズの完了条件。
8. `boss_schedule.json` — ボス解禁の初期スケジュール。
9. `mission_candidates.json` — 126件のミッション候補プール。

## AIへの重要指示
- `confirmed`は維持し、`draft`を確定仕様のように扱わない。`validation_required`は対象DLL／ソース／テスト環境で確認し、未確認APIを推測で使わない。
- 参加者コマンド不要、自動ログイン、SSC、匿名表示、既存Discord連携を変更しない、区域建築制限なし、報酬の重複／消失防止、未解禁ボスの全経路阻止は重要要件。技術的に満たせない場合は黙って省略しない。
- ゲーム内看板・ガイドNPC・ミッション端末、建築保護区域、自由建築区域の区分を追加しない。ルール・操作説明はDiscordに集約する。報酬の受取機能は初期スポーン位置のみ。
- 表彰10部門。公開ランキングや個人実績数は出さない。
- 既存Discord連携を改修したり、新しいDiscord botを暗黙に追加したりしない。
- 公開用 `docs/index.html` に内部API、SQLite、内部ID等を載せない。
- 各フェーズでビルド・テストし、未実施のテストをPASSと報告しない。

通常プラグイン本体の実装前に `anonymous_terraria_api_validation_matrix.md` のCriticalな確認項目を検証し、結果を `implementation_plan.md` に沿って反映してください。
