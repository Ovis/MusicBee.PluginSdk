# MusicBee.PluginSdk — Current State

最終更新: 2026-08-31

この文書は、MusicBee.PluginSdk の設計検討を別チャット・別作業環境から継続するための正本である。

## 目的

MusicBee 3.6 系向けPlugin開発を容易にする再利用可能なSDK / Frameworkを作る。

想定する最初の実利用Pluginは次の2つだが、両者は別Plugin・別Projectとして実装する。

1. Lyrics / Album Artwork 更新Plugin
   - MusicBee上では General Plugin
   - Lyrics更新後にArtwork更新などのアプリケーションロジックを持つ
   - HTTP / HTML解析 / 外部API / Cache / Rate Limit等は.NET 10 companion host側へ寄せる想定
2. Stream Deck 操作Plugin
   - Play/Pause/Next/Previous等をMusicBeeへ送る
   - TrackChanged / PlaybackState / Artwork等をStream Deck側へ返す
   - MusicBee側はnet48 bridge、外部側はmodern .NETまたはNode.js等を想定

## 基本アーキテクチャ

MusicBee本体が読み込む部分は .NET Framework 4.8 とする。

```text
MusicBee
  ↓
Generated MusicBee entrypoint
  ↓
MusicBee.PluginSdk.Framework
  ↓
IMusicBeePlugin / DI application services
  ↓ optional
MusicBee.PluginSdk.Ipc
  ↓ Named Pipe
Modern companion host (.NET 10 etc.)
```

SDKがMusicBeeに.NET 10 Plugin DLLを直接読み込ませることはしない。

## Package / Project構成

確定している構成:

- `MusicBee.PluginSdk.Api`
  - MusicBee APIのtyped wrapper
  - SDK-owned DTO / enum / value type
  - vendored `MusicBeeInterface.cs` は `Interop` 配下に隔離
- `MusicBee.PluginSdk.Framework`
  - lifecycle
  - DI
  - notifications / event dispatch
  - settings
  - dispatcher
  - framework bootstrap
- `MusicBee.PluginSdk.Generators`
  - MusicBee必須entrypointのSource Generator
  - metadata / startup contractのcompile-time validation
- `MusicBee.PluginSdk.Ipc`
  - optional
  - Named Pipe transport、framing、serialization、request/response/event、correlation、timeout、reconnection等
- `MusicBee.PluginSdk.Testing`
  - optional
  - fake client / fake feature APIs / notification helpers
- `MusicBee.PluginSdk`
  - metapackage
  - Api + Framework + Generatorsを参照
  - Ipc / Testingは明示追加

`Api`と`Framework`はnet48。SDK-style csprojを使用する方向。

## Public APIの設計原則

- MusicBee公式delegate、`IntPtr`、API revision等を通常のPublic APIへ漏らさない
- MusicBee固有概念はSDK-owned typeとして表現する
- facadeと細分化feature interfaceの両方を提供する
- MusicBee APIは同期APIとして公開する
- 隠れたTask化、隠れたthread switch、隠れたretryを避ける
- DTO / event snapshotは原則immutable
- Lazy Load / partial loaded DTOは作らない
- Advanced APIを別途用意してescape hatchを提供する
- normal APIでは検証済み機能のみ正式サポートする
- explicitnessを優先し、assembly scanningや命名規約依存のmagicは避ける
- AI/Codexが責務・extension pointを読み取りやすいAPIと文書構造にする

## Framework

### DI

- `Microsoft.Extensions.DependencyInjection`
- SDK serviceは原則Singleton
- Plugin実装はbase class継承ではなくinterface + DI
- Startupは `IMusicBeePluginStartup`
- `ConfigureServices(IServiceCollection services)` で明示登録

### Logging

- `Microsoft.Extensions.Logging`
- 主要なSDK障害にはstable Error Code
  - `MBSDK-API-*`
  - `MBSDK-FW-*`
  - `MBSDK-IPC-*`
- 主要lifecycle/state changeにはstable Event ID

### Lifecycle

Plugin側:

- `StartAsync(CancellationToken)`
- `StopAsync(CancellationToken)`

Framework側:

- MusicBeeの同期callbackとasync lifecycleをbridge
- Start中のnotificationはqueue
- Start成功後にhandler delivery開始
- Start失敗はFaulted、automatic restartなし
- Start/Stopにはframework timeoutを設ける
- timeout値はMusicBee実動作調査後に確定
- lifecycle stateはread-only public API
- `StateChanged`は同期.NET event

想定state:

`Created -> Starting -> Running -> Stopping -> Stopped`

失敗時は`Faulted`。

## Event model

2種類を併設する。

1. 軽量同期 `.NET event`
2. DI managed `IMusicBeeEventHandler<TEvent>`
   - `Task HandleAsync(TEvent, CancellationToken)`

同一event generationの複数DI handlerはparallel実行。
次generationは前generationのhandler完了後に開始する。

Eventはsemantic classを持つ。

### State Event

- 実行中handlerはcancelしない
- 同型eventがpendingしている場合は最新1件へcoalesce可能

例:

- TrackChanged
- PlayStateChanged
- SelectionChanged
- PlayingTracksChanged

### Occurrence Event

- silent dropしない
- bounded queue
- overflowは明示Fault / Error Code
- automatic retryなし

## Player / Now Playing

`IPlayer` facadeの下に意味のある単位でinterfaceを分離する。

- `IPlaybackController`
- `IPlaybackState`
- `IVolumeController`
- `IPlaybackModeController`
- `INowPlaying`
- `IPlaybackTimeline`

MusicBeeに直接primitiveがないPlay / Pause等を、state read + PlayPauseで擬似実装しない。

Player commandはMusicBee公式APIがboolしか返さないためPublic APIもbool。

`IsCommandAvailable(PlayerCommand)` を別途提供し、command前の自動precheckは行わない。

`INowPlaying.CurrentTrackId` は `TrackId?`。
`null`は正常な「current trackなし」。

`GetCurrentTrack()`は次の意味論:

1. 呼び出し開始後にcurrent `TrackId`を取得
2. そのIDの`TrackInfo` Snapshotを取得
3. 途中でcurrentが変わっても最初のTrackのSnapshotを返す

返却時点でもcurrentであることは保証しない。

`TrackChanged` payloadは新しい `TrackId?` のみ。
Previous Trackは載せない。
TrackChangedはState Eventとしてcoalesceする。

Timelineは`TimeSpan`を使用する方向。Position change eventは0.xでは提供せず、必要なPluginがpollする。

## Track model

### TrackId

SDK-owned opaque Value Type。
MusicBee内部でfile URL等をidentityとして使っていてもPublic APIへ露出しない。

### TrackLocation

SDK-owned Value Type。

- original MusicBee locationを保持
- local fileとは限らない
- `Kind`
- `IsLocalFile`
- `TryGetLocalPath()` 等を提供する方向

### TrackInfo

immutable point-in-time Snapshot。
getterがhidden API callを行わない。

基本FieldはSDK側で固定する。
利用頻度ではなく、MusicBee 3.6のBulk取得時にhidden per-track callを発生させず取得できるかを最優先してFieldを選定する。

DurationもBulk取得時に追加コストなく取れる場合だけ含める。

追加metadataは`IMetadata`等で明示取得する。

## Metadata

- individual read + multi-field read
- `MetadataField<T>`
- writable fieldは `WritableMetadataField<T>`
- normal APIは正式検証済みfieldのみ
- unsupported fieldはAdvanced API
- multi-valued fieldは検証済みの場合 `IReadOnlyList<string>`
- 順序・重複を保持
- MusicBee 3.6の明示的制約だけvalidate
- auto trim / clamp / normalizeしない
- `SetValue` と `RemoveValue` を分離

Immediate Metadata APIはauto-commitする。
write後のautomatic rereadは行わない。

## Track Edit Session

Metadata + Lyrics + Artworkを1 Track単位でstageする。

- Session creation時点がsingle baseline
- stage中はMusicBeeを書き換えない
- 同じtargetへの複数stageはlast operation wins
- commit前にedited targetのみoptimistic concurrency check
- unrelated field変更はconflictにしない
- any conflictならwrite開始前に全commitをabort
- `TrackEditConcurrencyException`
- conflict exception時はSDKによる変更ゼロを保証
- immediate APIにforce相当は持たせず、immediate APIそのものをunconditional overwriteとする
- commitはSDK固定順序
- exact orderはMusicBee動作検証後に確定
- unchanged staged valueはwriteせず`Unchanged`
- conflict detectionをUnchanged判定より先に行う
- writeはFail Fast
- first failure以後は`NotExecuted`
- automatic rollbackなし
- success時は必ず`TrackEditCommitResult`
- partial failureは`TrackEditCommitException`に同じResultを含める
- resultはper-change
- status: `Applied / Unchanged / Failed / NotExecuted`
- large payloadをresultへ複製しない
- Sessionはone-shot
- IDisposableではない
- thread-safeではない
- 同Trackに複数Session作成可
- stateをPublic APIとして公開
  - `Open / Committing / Committed / Failed`を候補

## Lyrics

- immutable `LyricsData`
- SDK-owned `LyricsType`
- LRC解析等は行わない
- provider name / source URL / retrieved time等はSDK core modelに含めない
- `ReplaceLyrics` / `RemoveLyrics`を分離
- null = missing、empty = present but empty をMusicBeeが区別できる場合のみPublic Contractにする
- write時LyricsType指定もMusicBeeが正式に対応している場合のみ公開

## Artwork

- immutable `ArtworkData`
- encoded binaryを保持し、SDK Coreでは画像decode / resize / conversionを行わない
- binaryは`ReadOnlyMemory<byte>`
- SDK取得時にbacking bufferを所有・固定する
- lightweight magic/signature validationを行う
- JPEG / PNGをformal support
- WebPはMusicBee 3.6で直接Artworkとして安定利用できるか実証後にformal support判断
- unsupported WebP等の変換はPlugin / companion host責務
- artwork size limitはsafe default + plugin configurable
- exact defaultは実測後決定
- SDK-owned `ArtworkType`
- 同一typeの複数ArtworkをMusicBeeが本当に保持できるか確認してAPI形状を決める

## Query / Selection

### TrackQuery

MusicBeeには`Library_QueryFilesEx(string query, out string[] files)`が存在し、auto-playlist XML queryを利用できる。

Public APIではverified field/comparisonだけをtyped queryとして公開する方向。

- immutable expression tree
- AND / OR / nesting
- unsupported queryは全件取得 + LINQまたはAdvanced API
- raw XMLはnormal APIへ露出しない

### TrackSort

TrackQueryとは別object。
複数key + priority order + ASC/DESC。

MusicBee側sortかSDK側sortかはAPI調査後に明示する。

### Query options

Limitはquery本体ではなくexecution option。
0.xでfake paginationは提供しない。

### Selection

`ISelection`を専用read-only APIとして提供。
Library queryのdomain特殊値としてPublic APIへ露出しない。

SelectionChanged payloadは`TrackId` listのみ。
State Eventとしてcoalesce。

## Playing Tracks / Queue

現在の再生キューはLive Collectionではなく、明示取得した時点のimmutable ordered Snapshotとして扱う。

`PlayingTrackEntry`を定義し、MusicBeeのqueue primitiveから追加Metadata取得なしで得られるqueue固有情報だけを保持する。

Track Title / Artist等のmetadataは含めない。

Current位置はEntryの`IsCurrent`ではなくSnapshot全体の`CurrentIndex`として表現する。

`PlayingTracksChanged` Eventは変更通知のみ。Snapshot全体をpayloadへ自動添付しない。
必要なHandlerが`GetSnapshot()`で明示取得する。
State Eventとしてcoalesceする。

## IPC

Core FrameworkはIPCを必須責務にしない。
`MusicBee.PluginSdk.Ipc`をoptional common libraryとして提供する。

### Transport

- transport abstractionを持つ
- 初期実装はNamed Pipeのみ
- server/client両方提供し、どちら側をserverとするかSDKでは固定しない
- current Windows userのみ接続可能なACL
- physical resource名はPlugin IDからSDKがpurpose-specific namespace/prefixを付与

### Protocol

- Request / Response / Eventを標準化
- explicit string Message ID
- message ID重複を検出
- envelope候補:
  - protocolVersion
  - kind
  - messageType
  - correlationId
  - payload
- Protocol Version handshake / validation
- incompatible versionはreject
- capability negotiationは初期実装なし
- JSON default + serializer abstraction
- binary attachmentを0/1個サポート
- binaryは初期実装`byte[]` full buffer
- size limitを設ける

### Reliability

- request timeout / cancellation
- reconnect mechanism
- disconnect時にin-flight messageをautomatic resendしない
- Connected / Disconnected / Reconnected notification
- concurrency policyを設定可能
  - Concurrent
  - Sequential
  - MaxConcurrency=N

### Companion process

optional `ICompanionProcessManager`。

- start
- duplicate prevention
- detect exit
- graceful shutdown
- restart/backoff/crash-loop suppression mechanism
- restart policyはPluginが明示指定
- default Never

## Settings

- `IPluginSettingsStore`
- standard JSON provider
- typed options layer
- Testingではin-memory provider
- `IPluginDataPathProvider`
- default `%AppData%`
- portable/plugin-local override可能
- Pluginがpathを手組みしない

Schema Version + explicit migration mechanismを持つ。
Migration時のみbackupを保持。
想定処理:

`backup -> migrate -> temp write -> atomic replace`

migration failureでsilent resetしない。

## Plugin identity / metadata

Plugin metadataはAssembly Attribute + Startup。

Stable Plugin IDをmetadataで必須にする。
Plugin IDはreverse-domainを強制せず、非empty safe ASCII stringとする。
具体regexは実装前に確定するが `[A-Za-z0-9._-]` 系を想定。

1 Plugin DLL = 1 MusicBee Plugin Type。

0.xでformal supportするPlugin Type:

- General
- LyricsRetrieval
- ArtworkRetrieval

特殊Plugin Typeは専用interfaceで表現し、Source Generatorがmetadataとの整合性をcompile-time validationする。

## Source Generator

MusicBeeが要求する `MusicBeePlugin.Plugin` entrypointを生成する。

Generatorは静的に判定可能な設定不備を積極的にdiagnosticする。

- required metadata
- Plugin ID format
- Startup count/type
- Plugin Typeと専用interfaceの整合
- known invalid configuration

Diagnosticは日本語かつactionableにする。

## Build / Packaging

- NuGet配布を最初から行う
- dependencyはPlugin DLLへembedする方針
- embeddingはSDK MSBuild Targetsで標準化
- explicit opt-out / exclusionを用意
- local MusicBee deployment用opt-in MSBuild Target
- MusicBee install pathはuser-level MSBuild property / environment config
- repoへlocal pathをcommitしない
- deploy requestedなのにpath未設定なら明示build error
- silent auto-detectは初期実装しない

Dependency embeddingの具体ツール（Costura.Fody等）は実証後決定。

## Compatibility / Quality

- MusicBee 3.6 seriesを初期support target
- official `MusicBeeInterface.cs`をvendored sourceとして固定
- source URL / acquisition date / API revision / local modificationsを記録
- scheduled CIでupstream diffを検知
- auto-updateはしない
- Public API baselineをrepoに保持しCIでbreaking change検知
- 0.xで実Pluginをdogfoodし、extension pointsが安定後1.0

## Documentation policy

### main branch

Plugin implementer / AI向け。

- README
- AGENTS.md
- plugin authoring guide
- extension points
- API usage
- generator usage
- focused samples
- XML documentation

### orphan `docs` branch

SDK自体の設計・開発管理用。

- current-state.md
- architecture documents
- design decisions
- research results
- implementation progress
- unresolved issues
- roadmap
- upstream tracking

## Code comments

- 日本語
- 常体
- classと主要public/internal methodは原則XML documentation comments
- 「何をしているか」より設計意図・理由を記録
- concurrency / ordering / compatibility / safety / lifecycle / retry / external API constraints等を積極的にコメント
- 自明な逐語訳コメントは避ける

## Official Samples予定

- Minimal General Plugin
- Notification subscription
- Player operations
- Lyrics / Artwork Provider
- IPC Bridge

## Research required / 条件付き未確定

実装前または該当API設計時に確認する。

1. MusicBee Plugin callback / APIのthreading constraints
2. MusicBee shutdown中の同期callbackとasync Start/Stop bridgeの安全な実装
3. Start / Stop timeout default
4. async event handler shutdown時のdrain / cancellation semantics
5. event bounded queue default size / Fault recovery
6. TrackInfoでBulk取得できる正確なField一覧
7. TrackQueryで正式対応可能なfield/comparison matrix
8. TrackSortをMusicBee側で効率的に実行できる範囲
9. Volumeの正式range / precision
10. Timeline Position / Durationの「値なし」表現とSeekの戻り値
11. TrackEdit commitの安全な固定適用順
12. Lyrics missing vs emptyをMusicBeeが区別できるか
13. Lyrics typeをwrite時に指定可能か
14. Artwork WebP direct support
15. Artwork type/index modelで同type multiple imageが実在するか
16. Artwork sizeの現実的分布とdefault size limit
17. IPCのnet48向けJSON serializer依存関係
18. exact IPC framing / handshake / cancellation protocol
19. companion host配布・探索path
20. vendored MusicBeeInterface.csのlicense / redistribution条件
21. x86 / AnyCPU等platform target
22. nullable / C# language version / analyzer set
23. Roslyn Generator target
24. dependency embedding tool
25. NuGet publish workflow / versioning
26. Error Code / Event ID catalog

## 現在地

設計質問Q1〜Q190まで完了。

直近の確定事項:

- Q187: Playing Tracksはordered Snapshot
- Q188: queue entryは`PlayingTrackEntry`
- Q189: current位置はSnapshot全体の`CurrentIndex`
- Q190: `PlayingTracksChanged`は通知のみでSnapshot payloadを持たない

次の設計質問は **Q191以降** とする。

ただしこのdocs退避作業自体を会話上Q191として扱っていたため、設計Decision番号の衝突を避けるなら次の技術設計質問を **Q192** から再開するのが安全である。
