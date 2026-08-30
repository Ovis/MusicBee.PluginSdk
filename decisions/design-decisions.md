# Design Decisions

この文書は、MusicBee.PluginSdk のdig設計で確定したDecisionをQ番号順に記録する。

詳細な現行仕様は `../current-state.md` を正本として参照する。

## Q1-Q20: Scope / Package / Core API

- **Q1 — Plugin Frameworkまで作る**
  - Official API hiding、初期化/終了、PluginInfo、notifications、facade、SDK-owned DTO/enums、plugin-type support、testable interfacesまで含める。
- **Q2 — Core + Framework分割**
  - `MusicBee.PluginSdk.Api` と `MusicBee.PluginSdk.Framework`を基本分割とする。
- **Q3 — SDK-owned public types中心**
  - Official delegate / IntPtr / API revision等は通常APIへ漏らさない。
- **Q4 — MusicBee APIは同期API**
  - 全APIをTaskで包まない。
- **Q5 — 主要APIを広くwrap**
  - Player / NowPlaying / Library / Metadata / Lyrics / Artwork / Notifications / Playlist / PlayingTracks / Volume / Playback modes / Settings / UI/Menu等。
- **Q6 — DI + interface中心**
  - base class継承は避け、MusicBee entrypoint shellだけを最小化する。
- **Q7 — DIはMicrosoft.Extensions.DependencyInjection**。
- **Q8 — LoggingはMicrosoft.Extensions.Logging abstractions**。
- **Q9 — SDK名はMusicBee.PluginSdk**。
- **Q10 — NuGet前提で開始**。
- **Q11 — MusicBeeInterface.csはvendored fixed upstream sourceとしてInterop隔離**。
- **Q12 — Api / Frameworkともnet48**。
- **Q13 — Core FrameworkはIPCを必須責務にしない**。
- **Q14 — optional small common IPC libraryを持つ**。
- **Q15 — Named Pipe標準 + transport abstraction**。
- **Q16 — IPC server/client側をSDKで固定しない**。
- **Q17 — Request/Response/Event message modelを標準化**。
- **Q18 — JSON default + serializer abstraction**。
- **Q19 — Protocol Version handshake/validationを持つ**。
- **Q20 — aggregate facade + feature interfaces両方を提供**。

## Q21-Q40: Events / Lifecycle / Generator / API Model

- **Q21 — .NET event + DI Event Handler併設**。
- **Q22 — null/Try/Exceptionを意味論に応じて使い分ける**。
- **Q23 — normal lifecycle + optional interfaces**。
- **Q24 — MusicBee entrypointはSource Generatorで生成**。
- **Q25 — metadata Attribute + Startup**。
- **Q26 — Startupは`IMusicBeePluginStartup` interface**。
- **Q27 — MusicBee SDK servicesはSingleton**。
- **Q28 — APIはthread switchしない。Frameworkがmarshal mechanismを提供**。
- **Q29 — `IMusicBeeDispatcher`を提供**。
- **Q30 — 初期support targetはMusicBee 3.6 series**。
- **Q31 — mainは利用者/AI向け、orphan docs branchはSDK設計/進捗向け**。
- **Q32 — focused sampleを複数用意**。
- **Q33 — conventionよりexplicitness優先**。
- **Q34 — Public XML docsにcontract/constraintsを書く**。
- **Q35 — internal class/major methodsも日本語XML docs**。
- **Q36 — `MusicBee.PluginSdk.Testing`を提供**。
- **Q37 — common `TrackInfo` + full Metadata API**。
- **Q38 — read DTO/events immutable、update modelは用途限定mutable**。
- **Q39 — raw escape hatchはAdvanced APIへ隔離**。
- **Q40 — scheduled CIでupstream interface差分検知のみ。auto-updateしない**。

## Q41-Q60: Compatibility / Packaging / IPC / Settings

- **Q41 — Public API baseline + CI**。
- **Q42 — 0.xで実Pluginをdogfoodしてから1.0**。
- **Q43 — `MusicBee.PluginSdk` metapackageを用意**。
- **Q44 — `dotnet new` templateを1つ用意**。
- **Q45 — opt-in MSBuild TargetでMusicBee Plugins directoryへdeploy**。
- **Q46 — MusicBee pathはuser-level MSBuild/environment config**。
- **Q47 — Plugin dependencyはPlugin DLLへembedする方針**。
- **Q48 — dependency embeddingをSDK MSBuild Targetsで標準化**。
- **Q49 — optional companion process managerをIpcへ提供**。
- **Q50 — restart mechanismは提供、policyはPluginが明示。default Never**。
- **Q51 — disconnect時にautomatic resendしない**。
- **Q52 — Named Pipe ACLはcurrent Windows user only**。
- **Q53 — IPC concurrency policyをconfigurableにする**。
- **Q54 — message identityはexplicit string Message ID**。
- **Q55 — JSON envelope + optional single binary attachment**。
- **Q56 — binary attachmentは初期`byte[]` full-buffer**。
- **Q57 — Source Generatorは静的に判定可能なcontractを積極的にdiagnostic**。
- **Q58 — major runtime errorsにstable Error Code**。
- **Q59 — major lifecycle/state changesにstable Event ID**。
- **Q60 — `IPluginSettingsStore` + standard JSON provider**。

## Q61-Q80: Identity / Migration / Event Execution / Lifecycle

- **Q61 — `IPluginDataPathProvider` + default `%AppData%`**。
- **Q62 — stable string Plugin IDをmetadataで必須化**。
- **Q63 — Plugin IDはarbitrary non-empty string。reverse-domain強制なし**。
- **Q64 — Plugin ID文字はsafe ASCII only**。
- **Q65 — physical resource namespaceはSDKがpurpose-specific prefixを付与**。
- **Q66 — settings schema version + explicit migration**。
- **Q67 — migration時のみbackupを保持**。
- **Q68 — DI Event Handlerはasync標準**。
- **Q69 — 同一event generationの複数handlerはparallel**。
- **Q70 — successive event generationsはsequential**。
- **Q71 — queue policyはState/Occurrence semantic classで分ける**。
- **Q72 — State eventは実行中1件 + pending latest 1件へcoalesce**。
- **Q73 — Occurrence overflowはbounded queue + explicit Fault**。
- **Q74 — handler automatic retryなし**。
- **Q75 — Plugin lifecycleは`StartAsync` / `StopAsync`**。
- **Q76 — StartAsync中eventはqueue、成功後delivery**。
- **Q77 — StartAsync failureのautomatic plugin restartなし**。
- **Q78 — StopAsyncにframework timeout**。
- **Q79 — StartAsyncにもframework timeout**。
- **Q80 — lifecycle stateはread-only public API**。

## Q81-Q100: Plugin Types / Editing / Track Model

- **Q81 — Lifecycle StateChangedは同期.NET eventのみ**。
- **Q82 — special Plugin Typesはdedicated interfaces**。
- **Q83 — 0.x formal plugin typesはGeneral / LyricsRetrieval / ArtworkRetrieval**。
- **Q84 — 1 Plugin DLL = 1 MusicBee Plugin Type**。
- **Q85 — immediate update API + staged edit session**。
- **Q86 — edit sessionはSDK内でstageし、Commit前にMusicBeeへ触らない**。
- **Q87 — Commit適用順はSDK固定**。
- **Q88 — partial commit failureはstructured result付きexception**。
- **Q89 — automatic rollbackなし**。
- **Q90 — optimistic concurrencyはedited fieldsのみcheck**。
- **Q91 — edit sessionにforce optionなし**。
- **Q92 — conflictは`TrackEditConcurrencyException`**。
- **Q93 — immediate APIはunconditional updateのみ**。
- **Q94 — batch editingはSDK coreで扱わず1-track primitivesのみ**。
- **Q95 — `TrackId` SDK-owned opaque Value Type**。
- **Q96 — identityとlocationを分離**。
- **Q97 — locationは`TrackLocation` Value Type**。
- **Q98 — `TrackInfo`はimmutable point-in-time Snapshot**。
- **Q99 — `TrackInfo`はlightweight basic infoのみ**。
- **Q100 — library track enumerationは`IReadOnlyList<TrackInfo>` bulk**。

## Q101-Q120: Query / Artwork / Lyrics

- **Q101 — typed TrackQueryを可能な範囲で採用。無理ならbasic fetch+LINQへfallback**。
- **Q102 — typed TrackQueryはverified field/comparisonだけ正式サポート**。
- **Q103 — TrackQueryはAND/OR + nesting**。
- **Q104 — TrackQueryはimmutable**。
- **Q105 — sortingは`TrackSort`としてqueryと分離**。
- **Q106 — TrackSortはmultiple keys + priority order**。
- **Q107 — Limitはexecution option**。
- **Q108 — 0.xでpagingを提供しない**。
- **Q109 — Artworkはencoded binary `ArtworkData`**。
- **Q110 — binary ownershipは`ReadOnlyMemory<byte>`**。
- **Q111 — lightweight signature/format validation**。
- **Q112 — JPEG/PNG formal support。WebPはMusicBee direct support確認後判断**。
- **Q113 — unsupported WebP conversionはPlugin/companion host責務**。
- **Q114 — Artwork size limitはsafe default + plugin configurable**。
- **Q115 — SDK-owned `ArtworkType` enum**。
- **Q116 — same-type multiple ArtworkはMusicBee実モデル確認後、可能ならCollection、不可ならone-per-type**。
- **Q117 — write APIはmultiplicity実態に合わせReplace/AddまたはReplace-only**。
- **Q118 — Lyricsはimmutable `LyricsData`**。
- **Q119 — missing vs emptyはMusicBeeが区別できる場合のみ区別**。
- **Q120 — `ReplaceLyrics` / `RemoveLyrics`を分離**。

## Q121-Q140: Lyrics / Metadata / Edit Result

- **Q121 — Lyrics type write指定はMusicBeeが正式対応する場合のみ**。
- **Q122 — provider/source metadataはLyricsDataに含めない**。
- **Q123 — SDK-owned `LyricsType` enum**。
- **Q124 — Metadata readはindividual + multi-field**。
- **Q125 — typed `MetadataField<T>`**。
- **Q126 — unsupported metadata fieldはnormal API外**。
- **Q127 — writable fieldは`WritableMetadataField<T>`**。
- **Q128 — null/empty distinctionはMusicBeeが区別するfieldのみ**。
- **Q129 — metadata removalは`SetValue`と`RemoveValue`を分離**。
- **Q130 — verified multi-value fieldは`IReadOnlyList<string>`**。
- **Q131 — multi-value orderを保持**。
- **Q132 — duplicatesを保持**。
- **Q133 — explicit MusicBee constraintsのみvalidate。auto-correctなし**。
- **Q134 — immediate Metadata APIはauto-commit、Edit Sessionはfinal commit一回**。
- **Q135 — write後automatic rereadなし**。
- **Q136 — TrackEditSession scopeはMetadata + Lyrics + Artwork**。
- **Q137 — same targetへ複数stageした場合last operation wins**。
- **Q138 — unchanged staged valueはNo-opとしてwriteしない**。
- **Q139 — No-op result statusは`Unchanged`**。
- **Q140 — normal commit successも必ず`TrackEditCommitResult`を返す**。

## Q141-Q160: Edit Semantics / Selection / Track Events

- **Q141 — commit result granularityはper change**。
- **Q142 — write failureはFail Fast**。
- **Q143 — conflict detectionをUnchanged判定より先に実施**。
- **Q144 — any conflictでcommit全体をwrite前abort**。
- **Q145 — session creation時がsingle baseline time**。
- **Q146 — session expirationなし**。
- **Q147 — TrackEditSessionはIDisposableにしない**。
- **Q148 — Sessionはone-shot**。
- **Q149 — post-commitでもstaged contentをread-only参照可能**。
- **Q150 — public `TrackEditSessionState`**。
- **Q151 — TrackEditSessionはthread-safeではない**。
- **Q152 — 同一Trackに複数Session作成可**。
- **Q153 — conflict detailは軽量metadata actual value、Lyrics/Artworkはfingerprint/length/type/format**。
- **Q154 — selectionはdedicated read-only `ISelection`**。
- **Q155 — selection APIは0.xではread-only**。
- **Q156 — SelectionChanged payloadはTrackId listのみ**。
- **Q157 — SelectionChangedはState Event/coalescing**。
- **Q158 — TrackChanged payloadはTrackIdのみという方針**。
- **Q159 — TrackChangedはState Event。履歴/Scrobble的Occurrenceは別概念**。
- **Q160 — PlayStateChanged payloadにnew PlaybackStateを含める**。

## Q161-Q180: Player / Timeline / NowPlaying

- **Q161 — `PlaybackState`はMusicBee 3.6の明確にobservableな全stateをSDK enumへmap**。
- **Q162 — `IPlayer` facade + `IPlaybackController` / `IPlaybackState`**。
- **Q163 — Player feature splitはController/State/Volume/PlaybackMode程度の意味単位**。
- **Q164 — Play/Pause等はMusicBee direct primitiveのみ。擬似commandを作らない**。
- **Q165 — structured command resultはMusicBeeにreasonがある場合のみ、という条件を調査**。
- **Q166 — 現行MusicBeeはboolのみなのでPublic Player command resultもbool**。
- **Q167 — command前automatic enabled precheckなし**。
- **Q168 — availability APIは`IsCommandAvailable(PlayerCommand)`**。
- **Q169 — command availability eventなし**。
- **Q170 — Volume public unitはMusicBee formal precision/range調査後確定**。
- **Q171 — MuteはVolume=0とみなさず独立state、`IVolumeController`内**。
- **Q172 — Volume/Mute changeはMusicBee direct NotificationのみEvent化**。
- **Q173 — Shuffle/RepeatはSDK-owned typed enumsで全verified mode保持**。
- **Q174 — Shuffle/Repeat changeもdirect MusicBee NotificationのみEvent化**。
- **Q175 — `INowPlaying`と`IPlaybackTimeline`を分離**。
- **Q176 — timeline time typeは`TimeSpan`**。
- **Q177 — SeekもMusicBee direct primitiveのみ**。
- **Q178 — Position changed eventは0.xで提供しない**。
- **Q179 — current track absentは`TrackId? CurrentTrackId`のnull**。
- **Q180 — convenience `INowPlaying.GetCurrentTrack()`を提供**。

## Q181-Q190: NowPlaying Consistency / Queue

- **Q181 — `GetCurrentTrack()`は最初に取得したTrackIdを固定し、そのTrackのSnapshotを返す**。
  - 途中でcurrentが変わってもretryしない。
  - 返却時点でcurrentであることは保証しない。
- **Q182 — TrackChanged payloadは`TrackId?`**。
  - nullでcurrent trackなしを表現。
- **Q183 — TrackChangedにはPreviousTrackIdを含めない**。
- **Q184 — TrackInfo.DurationはBulk取得で追加コストなく取れる場合のみ含める**。
- **Q185 — TrackInfo field選定は利用頻度よりBulk取得コスト優先**。
- **Q186 — TrackInfo基本field setはSDK側で固定**。
- **Q187 — Playing Tracks / Queueはordered immutable Snapshot**。
- **Q188 — queue elementは`PlayingTrackEntry`**。
  - queue primitiveから追加Metadata取得なしで得られるqueue固有情報のみ。
- **Q189 — current位置はEntryの`IsCurrent`ではなくSnapshot全体の`CurrentIndex`**。
- **Q190 — `PlayingTracksChanged`は変更通知のみ**。
  - Snapshot payloadを持たない。
  - 必要なhandlerが明示的に`GetSnapshot()`する。
  - State Eventとしてcoalesce。

## 条件付きDecision

以下は方針は決まっているが、MusicBee 3.6の実API/挙動確認で最終Public Contractが確定する。

- Q101 TrackQuery typed coverage
- Q112 WebP artwork support
- Q116/Q117 Artwork multiplicity/write shape
- Q119 Lyrics missing vs empty
- Q121 Lyrics type write support
- Q170 Volume range/precision
- Q184 TrackInfo Duration inclusion

## 次の番号

Q191はdocsへの退避自体の確認に使用したため、次の技術設計Decisionは **Q192** から開始する。
