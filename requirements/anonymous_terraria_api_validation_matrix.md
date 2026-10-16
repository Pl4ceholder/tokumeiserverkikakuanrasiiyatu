# TShock API事前調査マトリクス

- 調査日：2026-10-16
- 基準候補：TShock 6.2.1 / Terraria 1.4.5.8 / OTAPI 3.3.14 / TShockAPI TargetFramework net9.0
- `source-confirmed`は当該ソース上に型・イベントが存在すること、`prototype-required`は要件に合う挙動を確認していないこと、`blocker`は解消するまで対象機能を完成扱いしないことを示す。

## 公開ソースで確認した事実

| 項目 | 確認内容 | 未確認／注意点 | 状態 | 参照先 |
|---|---|---|---|---|
| バージョン | 公式リリース一覧ではTShock 6.2.1 for Terraria 1.4.5.8が最新として掲載。リリースノートにOTAPI 3.3.14、TShockAPI.csprojにnet9.0。 | 実装開始時に同バージョンが最新か、導入端末の.NETランタイムが一致するか。 | source-confirmed / deployment-recheck | https://github.com/Pryaxis/TShock/releases ; https://raw.githubusercontent.com/Pryaxis/TShock/v6.2.1/TShockAPI/TShockAPI.csproj |
| タイル編集 | `GetDataHandlers.TileEdit`があり、X/Y/Action/EditData/editDetail/Styleを含む。 | 要求イベントのタイミング、失敗操作、成功後のワールド状態、爆発等。 | prototype-required | https://raw.githubusercontent.com/Pryaxis/TShock/v6.2.1/TShockAPI/GetDataHandlers.cs |
| 複数タイルオブジェクト | `GetDataHandlers.PlaceObject`がオブジェクト配置用イベントとして存在。TShock Bouncerのソースは複数タイル物体が通常のOnTileEdit経由ではないケースを示す。 | ブロック・壁・足場・トーチ・家具の計数で二重計上や漏れがないか。 | prototype-required | https://raw.githubusercontent.com/Pryaxis/TShock/v6.2.1/TShockAPI/GetDataHandlers.cs ; https://github.com/Pryaxis/TShock/blob/v6.2.1/TShockAPI/Bouncer.cs |
| スポーン報酬チェスト | `ChestOpen`は座標とプレイヤーを持ち、Handled可能なイベントとして定義。`PlaceChest`も存在。 | vanilla通常操作を確実に止めるか、対象チェストの破壊・移動、同時操作での正確さ。 | prototype-required | https://raw.githubusercontent.com/Pryaxis/TShock/v6.2.1/TShockAPI/GetDataHandlers.cs |
| 釣り | `FishOutNPC`はNPCを釣り上げるイベント。 | 通常の魚・アイテム・ジャンク・クエスト魚を成功として数える汎用イベントはGetDataHandlers.cs内では確認できなかった。 | blocker | https://raw.githubusercontent.com/Pryaxis/TShock/v6.2.1/TShockAPI/GetDataHandlers.cs |
| ログイン | `/login`の引数なしUUID経路は現キャラクター名に対応するユーザーアカウントを取得。任意ユーザー名＋パスワードは`AllowLoginAnyUsername`依存。 | 内部ID・自動アカウント生成・自動ログイン・SSC初期化・再接続の順序。 | prototype-required | https://raw.githubusercontent.com/Pryaxis/TShock/v6.2.1/TShockAPI/Commands.cs |
| ボス制御 | `HandleSpawnBoss`はクライアントからのボス／侵攻要求パケットを扱う。TSAPIにはキャンセル可能なNPC Spawn hookソースがある。 | 対象パッケージのフック版、自然スポーン、イベント、変身／派生出現、Lunar eventまでの全カバレッジ。 | blocker | https://raw.githubusercontent.com/Pryaxis/TShock/v6.2.1/TShockAPI/GetDataHandlers.cs ; https://github.com/Pryaxis/TSAPI |
| 戦闘参加 | `NPCStrike`、`ProjectileKill`等のイベントが存在。 | メレー／投射物／召喚ミニオン／DoTを含めて有効な参加を一貫して判定する方法。 | prototype-required | https://raw.githubusercontent.com/Pryaxis/TShock/v6.2.1/TShockAPI/GetDataHandlers.cs |
| 開花ハーブ | タイルイベントから座標・編集内容を得られる。 | タイルID・frame情報で7種の開花状態を区別し、採集成功後に一度だけ加算できるか。 | prototype-required | https://raw.githubusercontent.com/Pryaxis/TShock/v6.2.1/TShockAPI/GetDataHandlers.cs |
| 富豪 | SSCキャラクターのアイテム状態を参照する方向。 | 個人保管庫のコインを読むAPI、表彰時点の一貫したスナップショット。 | prototype-required | https://github.com/Pryaxis/TShock |

## 実装前の最小技術スパイク

1. TShock 6.2.1／Terraria 1.4.5.8／.NET 9で空プラグインをビルド・ロードし、参照DLLを固定する。
2. 内部アカウントを作成し、接続時の自動ログインとSSC初期化を検証する。参加者の手入力なしに参加でき、再接続で同じキャラクターが戻ること。
3. 指定座標の`ChestOpen`をHandledにし、通常チェスト動作が止まること。2人同時クリック、連打、切断時も各人1日1回で、周辺建築は自由であること。
4. `TileEdit`＋`PlaceObject`で通常ブロック、壁、足場、トーチ、家具を置く／壊し、採掘・建築・ハーブ集計の重複と漏れを確認する。
5. 通常の魚、箱、ジャンク、クエスト魚、釣りNPCを試し、釣果成功を正確に検出する。できなければ釣り神の実装可否を未解決のままにする。
6. ボスごとに召喚アイテム、自然スポーン条件、イベント、NPC変身／派生、クライアントのspawn要求、Lunarイベントをテストし、未解禁ボスが一つも出現しないこと。
7. ボス戦に召喚者、近接、投射物、召喚武器使用者、AFK参加者を用意し、皆勤賞が定義どおり参加／非参加を判定すること。
8. 報酬付与の前後でサーバーを強制終了し、SSCデータとプラグインDBの間の報酬重複／欠落を検出し、安全に復旧できること。
9. サーバー停止中に日次／ボス解禁時刻を跨ぎ、再起動後に現時点へ同期して二重付与・再ロックがないこと。

## ブロッカーの扱い
- 通常釣果を確実に取得できない場合、`釣り神`を通常アイテム取得数へ勝手に置き換えない。実現手段が見つかるまで実装ブロッカーとして記録する。
- 未解禁ボスの全出現経路を阻止できなければ、ボス制御を完成扱いしない。
- 自動ログインとSSCの組み合わせ／報酬クラッシュ整合性が成立しない場合も、要件を黙って省略せず方式を再設計する。
