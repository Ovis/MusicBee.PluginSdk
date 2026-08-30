# MusicBee 3.6 Design Audit — 2026-08-31

この文書は、Q1〜Q201までに行った設計をMusicBee 3.6の実API・実挙動と照合する監査のチェックポイントである。

## 監査方針

Q202で、新規設計を一旦止めてQ1〜Q201を体系的に監査する方針を確定した。

今後は原則として次の順序で設計する。

1. 対象機能のMusicBee 3.6公式`MusicBeeInterface.cs`を確認する
2. 公式ドキュメントを確認する
3. 不明点はMusicBee公式Forumや現行Plugin実装例を調査する
4. それでも不明なら実機検証項目として残す
5. 確認できた実態を前提としてSDK Public Contractを設計する

既存Decisionについても、実態と矛盾する場合はQ番号を保持したまま補正・撤回・条件変更を記録する。

監査状態は必要に応じて `Verified` / `SDK design` / `Needs correction` / `Needs test` / `Superseded` 等で管理する。

## Baseline

- 対象: MusicBee 3.6系
- 公式interface: `https://getmusicbee.com/download/plugins/MusicBeeInterface.cs`
- 表向きの対応表記はMusicBee 3.6とする
- 技術的な互換性境界はInterface Version + API Revisionで管理する
- SDKが要求する最低API Revisionの具体値は、監査完了後に通常Public APIが必要とする最も新しいPrimitiveから決定する

## Q1〜Q30監査

### 概要

Package分割、DI、同期API、SDK-owned type、Source Generator等のSDK内部設計について、MusicBee実APIと直接矛盾する事項は確認されていない。

一方、Threading、Lifecycle、Version compatibilityについて補正した。

### Threading — Q28/Q29をQ203で補正

従来:

- API自体はthread switchしない
- Frameworkがmarshal mechanismを提供する
- `IMusicBeeDispatcher`を提供する

監査後:

- `IMusicBeeDispatcher`は廃止する
- MusicBee NotificationはGUI threadとは別threadから呼ばれる場合がある
- Notification callback threadからMusicBee APIを呼ぶこと自体は可能というMusicBee作者の説明がある
- FrameworkはMusicBee API呼び出しを特定threadへ自動marshalしない
- MusicBeeに存在しないthread-affinity contractをSDK側で作らない
- UI等、本当にthread affinityが確認された機能だけを、そのAPI設計時に個別検討する

### Lifecycle startup — Q204/Q208

MusicBeeの`Initialise()`と`PluginStartup` NotificationをFramework lifecycleへ次のようにmapする。

`Initialise()`:

- MusicBee API interfaceを受領
- compatibility検証
- SDK/DI container構築
- Plugin instance生成
- Notification受信準備
- `StartAsync`は呼ばない

`PluginStartup`:

- Framework stateを`Starting`へ遷移
- Framework管理Taskとして`StartAsync(CancellationToken)`を開始
- MusicBeeのNotification callback自体は`StartAsync`完了を待たず速やかにreturnする
- 成功時`Running`
- timeout/exception時`Faulted`
- automatic restartなし

MusicBee作者も実処理を`Initialise()`ではなくsuccessful initialization後の`PluginStartup`側へ置くことを案内している。

### Lifecycle shutdown — Q209

MusicBee側の終了entrypointは同期`Close(PluginCloseReason reason)`である。

確認した`PluginCloseReason`:

- `MusicBeeClosing`
- `UserDisabled`
- `StopNoUnload`

`Close()` return後もPluginの非同期cleanupを安全に継続できる保証は確認できないため、Frameworkは次の契約とする。

- `Close()`受信で`Stopping`へ遷移
- `StopAsync(CancellationToken)`を開始
- `Close()`はStopAsync完了またはFramework timeoutまで同期的に待つ
- timeout/exceptionはstable Error Code / loggingで明示する
- 無期限にはMusicBeeをblockしない
- `Close()` return後にcleanupが継続可能であることをPublic Contractとして保証しない

StartupとShutdownは意図的に非対称となる。

Start/Stop timeoutの具体値は実機検証後に確定する。

### Compatibility — Q30をQ205で補正

従来の「MusicBee 3.6 series support」に加え、実際のcompatibility contractをInterface Version + API Revisionで定義する。

- 初期化時にversion/revisionを検証する
- SDKの最低条件未満なら明示的に初期化を拒否する
- normal Public APIは最低revisionで保証可能な機能を基準にする
- 最低revisionの数値は監査完了後に決定する

## Q31〜Q60監査

Documentation、Testing、Advanced API、Public API baseline、IPC等の大部分はSDK designであり、MusicBee実APIとの重大な矛盾は現時点で確認されていない。

### Plugin deployment / dependency

MusicBeeが認識するPlugin本体DLLには`MB_*.dll`という命名・Plugins directory配置の制約がある。

SDKのMSBuild Targets / templateでは、これらのMusicBee固有制約をPlugin作者が毎回手作業で再現しなくてよいよう標準化する方向は維持する。

依存DLLのembed方式そのものはPackaging監査で引き続き検証する。

## Q61以降で確定した補正

### Persistent storage — Q61をQ206/Q207で補正

MusicBeeには`Setting_GetPersistentStoragePath()`が正式に存在する。

従来の`%AppData%` defaultを撤回し、次を標準とする。

- `IPluginDataPathProvider`のdefault rootは`Setting_GetPersistentStoragePath()`
- root直下にPlugin IDをそのままdirectory名として使用する
- Plugin IDはQ64のsafe ASCII制約によりfilesystem-safeにする
- PluginがMusicBeeの保存先policyを独自に再実装しない

## Playlist監査で判明した事項

### Playlist write primitives — Q198/Q201

公式interfaceで確認できるPlaylist操作には、少なくとも次がある。

- Create
- Delete
- Set/Replace files
- Append/Add files
- RemoveAt
- MoveFiles
- PlayNow

Renameの直接Primitiveは確認できない。

そのためSDK normal APIはMusicBeeに直接Primitiveが存在する操作だけを正式サポートし、Renameを合成して提供しない。

Playlist削除はQ200で、通常の明示的破壊操作として提供し、SDKがconfirmation UI/tokenを要求しないと確定した。

### Playlist contents — Q197補正

Playlist entry固有Metadataを返すPrimitiveは確認できない。

indexは`RemoveAt`/`MoveFiles`等の操作上の位置指定として意味を持つが、entry固有dataではない。このためPlaylist contentsのread modelは独自`PlaylistTrackEntry`を作らず、基本的にordered `IReadOnlyList<TrackId>`とする方向で扱う。

### Playlist notifications

公式Notificationには次が存在する。

- `PlaylistCreated`
- `PlaylistUpdated`
- `PlaylistDeleted`
- `PlaylistMoved`

Event payload/semantic classはNotification全体監査で確定する。

## 監査中に発見した未処理事項

### Playback mode

公式interface上、Shuffleはbool、Repeatは複数modeを持つ。

Q173の「Shuffle/RepeatをSDK-owned typed enumsで全verified mode保持」はShuffle側について過剰抽象化の可能性がある。Player監査で正式に補正する。

### Playing Tracks / Queue

公式interfaceにはNow Playing Listの取得・操作Primitiveが存在する。

またNotificationには`PlayingTracksChanged`とは別に`PlayingTracksQueueChanged`が存在する。

Q187〜Q192付近はこの差を十分確認せず設計していたため、Notification/Queue監査で再検討する。

### Notification/Event model

Q68〜Q74でState Event / Occurrence EventのFramework policyを設計したが、個々のMusicBee Notificationの意味・payload・発火条件を確認してから分類を再検証する必要がある。

次の監査対象ではMusicBee 3.6のNotificationを列挙し、次を確認する。

- Notification名
- 公式payload
- 発火条件
- 重複/連続発火の意味
- `PlayingTracksChanged`と`PlayingTracksQueueChanged`の差
- SDK Eventとして直接map可能か
- State / Occurrenceのどちらとして扱うべきか

## 次の作業

新規機能設計は引き続き一時停止する。

次はNotification/Event modelを実態監査し、その結果をQ68〜Q74、Q81以降、Q157〜Q174、Q187〜Q192等へ反映する。
