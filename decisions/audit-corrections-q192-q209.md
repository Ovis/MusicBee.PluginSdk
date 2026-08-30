# Design Decisions — Q192〜Q209 / Audit Corrections

この文書は`design-decisions.md`作成後に確定したDecisionと、MusicBee 3.6実態監査による補正を記録する。

## Q192〜Q201: Queue / Playlist

- **Q192 — CurrentIndexだけの専用Eventは作らない**
  - Track変更はTrackChanged、queue構造変更はPlaying Tracks系Notificationで扱い、CurrentIndexが必要な場合はSnapshotを明示取得する方向。
  - `PlayingTracksChanged` / `PlayingTracksQueueChanged`の実態監査後に再検証する。
- **Q193 — Queue mutation APIは0.x normal APIではread-only**
  - MusicBeeにmutation primitive自体は存在するため、Queue監査で再検証対象。
- **Q194 — SDK-owned opaque `PlaylistId` Value Type**。
- **Q195 — `PlaylistInfo`はimmutable point-in-time Snapshot**。
- **Q196 — `PlaylistInfo`にTrack listを含めない**。
- **Q197 — Playlist contentsはentry固有情報が実在する場合のみEntry型**。
  - 監査でentry固有Metadata primitiveが確認できなかったため、ordered `IReadOnlyList<TrackId>`方向へ補正。
- **Q198 — Playlistはbasic CRUDを提供**。
  - 監査でRename direct primitiveが存在しないことが判明したためQ201で補正。
- **Q199 — Playlist writeは即時操作**。
  - `PlaylistEditSession`は作らない。
- **Q200 — Playlist deleteは通常の明示的破壊メソッド**。
  - confirmation UI/tokenをSDKは要求しない。
- **Q201 — Playlist normal write APIはMusicBee direct primitiveが存在する操作だけ正式サポート**。
  - Create / Delete / Set/Replace files / Append/Add files / RemoveAt / MoveFiles / PlayNow等。
  - Renameは合成しない。

## Q202: Design audit

- **Q202 — 新規設計を一旦止め、Q1〜Q201をMusicBee 3.6実態と体系的に監査する**。
- 以後、新規Decisionも原則「実API調査 → 事実整理 → 設計判断」の順で進める。
- 既存Decisionの不整合はQ番号を保持したまま補正・撤回・条件変更する。

## Q203〜Q205: Threading / Lifecycle / Compatibility

- **Q203 — `IMusicBeeDispatcher`を廃止**。
  - Q28/Q29を補正。
  - MusicBee API呼び出し全般を特定threadへmarshalしない。
  - UI等で実際にthread affinityが確認されたAPIのみ個別設計する。
- **Q204 — `Initialise()`ではFramework/DI構築まで、`PluginStartup`で`StartAsync`を開始**。
- **Q205 — compatibility contractはMusicBee 3.6表記 + Interface Version/API Revision**。
  - 最低revisionの具体値は全監査後に決定する。

## Q206〜Q207: Persistent storage

- **Q206 — default data rootを`Setting_GetPersistentStoragePath()`へ変更**。
  - Q61の`%AppData%` defaultを撤回。
- **Q207 — Plugin IDをPersistent Storage Path直下のdirectory名としてそのまま使用**。
  - Q64のsafe ASCII制約を前提とする。

## Q208〜Q209: Async lifecycle bridge

- **Q208 — `PluginStartup` callbackは`StartAsync`完了を待たずreturn**。
  - Framework管理Taskとしてstartupを実行する。
  - 成功時Running、timeout/exception時Faulted。
- **Q209 — `Close()`では`StopAsync`をtimeout付きで同期的に待つ**。
  - `Close()` return後の非同期cleanup継続を保証しない。
  - timeout/exceptionは明示的にlogging/Error Codeへ記録する。
  - timeout具体値は実機検証後に確定する。

## 既存Decisionへの主要な補正一覧

| 元Decision | 状態 | 補正 |
|---|---|---|
| Q28 | Superseded by Q203 | MusicBee APIのthread marshal mechanismをFramework責務にしない |
| Q29 | Superseded by Q203 | `IMusicBeeDispatcher`廃止 |
| Q30 | Refined by Q205 | Interface Version/API Revisionを実compatibility境界に追加 |
| Q61 | Superseded by Q206/Q207 | `%AppData%`ではなくMusicBee Persistent Storage Path + Plugin ID |
| Q75 | Refined by Q204/Q208/Q209 | Start/StopのMusicBee callbackへの具体的mappingを確定 |
| Q78 | Refined by Q209 | Close内でStopAsyncをtimeout付き同期wait |
| Q79 | Refined by Q208 | StartAsyncはFramework管理Task、PluginStartup callbackはblockしない |
| Q197 | Refined by API audit | Playlist entry固有dataがないためTrackId ordered list方向 |
| Q198 | Superseded by Q201 | Renameを除外しdirect primitiveのみformal support |

## 未処理の監査候補

- Q68〜Q74: MusicBee Notification実態に基づくState/Occurrence分類
- Q76: Start中Notification queueの具体的対象
- Q83/Q84: special Plugin Type callback shape
- Q157〜Q174: Track/Player notification payloadと発火条件
- Q173: Shuffleがboolである実態への補正
- Q187〜Q193: `PlayingTracksChanged` / `PlayingTracksQueueChanged`およびNow Playing List primitiveに基づく再監査
