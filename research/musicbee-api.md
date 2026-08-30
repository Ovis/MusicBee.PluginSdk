# MusicBee API Research Notes

最終更新: 2026-08-31

この文書は設計Decisionではなく、MusicBee公式APIや周辺情報から確認済みの事実と、未検証事項を記録する。

## Baseline

- 初期support target: MusicBee 3.6 series
- 2026-08時点のMusicBee現行3.6系を前提に設計中
- Official developer API: `https://getmusicbee.com/help/api/`
- Official C# interface source: `https://getmusicbee.com/download/plugins/MusicBeeInterface.cs`
- `MusicBeeInterface.cs`はSDK側へvendored sourceとして固定し、Interop配下へ隔離する方針

## Runtime / Target Framework

MusicBeeが直接ロードするPlugin DLLは.NET Framework 4.x系を前提とする。
SDKのMusicBee-loaded側はnet48とする。

modern .NETが必要な処理はcompanion processへ分離し、Named Pipe等でbridgeする。

SDKはMusicBeeに.NET 10 Plugin DLLを直接ロード可能にするものではない。

## Current official interface facts already observed

### Player actions

Current official interfaceではPlayer action delegateは`bool`を返す。

確認済みの代表操作:

- PlayPause
- Stop
- PreviousTrack
- NextTrack
- PreviousAlbum
- NextAlbum

失敗理由を表すstructured resultは公式primitiveから得られない。
そのためSDK Public APIもboolとするDecisionになっている。

別にbutton enabled取得primitiveがあるが、SDKはcommand前のautomatic precheckを行わない。

### Play state

Current official interfaceで確認済みのPlayState:

- Undefined
- Loading
- Playing
- Paused
- Stopped

SDKではofficial enumを直接露出せずSDK-owned enumへmapする。

### Volume

Current official interfaceで確認済み:

- GetVolume returns `float`
- SetVolume accepts `float`

ただしsourceだけではformal range / precisionのcontractが明確ではない。
Public APIを0-100 intにするかfloat精度を保つかは未確定。

### Notifications

Current official interfaceで確認済みの代表Notification:

- TrackChanged
- PlayStateChanged
- TagsChanged
- RatingChanged
- FileAddedToLibrary
- PlaylistCreated
- PlaylistUpdated
- PlaylistDeleted
- SelectedFileChanged
- FileRenamed
- TempoChanged
- VolumeMuteChanged
- VolumeLevelChanged
- PlayerRepeatChanged
- PlayerShuffleChanged

SDKはMusicBeeに直接対応するNotificationを優先し、pollingや推測で擬似Eventを生成しない方針。

### Library query

Current official interfaceには次のprimitiveが存在することを確認済み。

`Library_QueryFilesEx(string query, out string[] files)`

MusicBee forum等の情報から、queryにはauto-playlist XML形式を利用できる。
`Conditions` / `Condition`を用いた条件式が存在し、domainとしてLibrary / SelectedFiles / DisplayedFiles等の概念も確認されている。

ただしfieldによって対応状況が一様ではない事例があるため、SDK typed `TrackQuery`は実証済みfield/comparisonだけを正式サポートする。

## Plugin Types

Official interfaceには多数のPlugin Typeがある。

初期0.xでSDKがformal supportするのは次の3種類に限定するDecision。

- General
- LyricsRetrieval
- ArtworkRetrieval

その他は実利用とAPI検証後に追加する。

## Entry point

MusicBee Pluginは`MusicBeePlugin.Plugin`型などMusicBeeが要求する固定entrypoint shapeを持つ。

SDKではこのboilerplateをSource Generatorで生成し、Plugin実装からInterop詳細を隠す。

## API compatibility

MusicBee Plugin APIはbackwards compatibilityを重視している。
ただしSDKでは公式sourceの最新版へ自動追従せず、vendored baselineを明示的に更新する。

scheduled CIでofficial sourceとの差分だけを検知する。

## Items requiring direct verification before Public Contract freeze

### Threading

- Initialise / Configure / SaveSettings / Close / Uninstall callback thread
- Notification callback thread
- API callをMusicBee UI threadへmarshalする必要がある範囲
- Plugin shutdown時にcallbackが再入する可能性

この結果で`IMusicBeeDispatcher`の実装契約を確定する。

### Async lifecycle bridge

- MusicBee同期initialise callbackをどこまでblock可能か
- StartAsync完了を待つ必要があるか
- shutdown時のStopAsync許容時間
- MusicBee process終了との競合

結果に基づいてStart/Stop timeout defaultを決める。

### TrackInfo bulk cost

各basic fieldがLibrary bulk primitiveだけで取得可能かを確認する。

特に:

- Title
- Artist
- Album
- AlbumArtist
- Track number
- Disc number
- Year
- Duration
- Rating
- Genre

per-track追加API呼び出しが必要なfieldは`TrackInfo`基本fieldから外す。

### TrackQuery support matrix

実際のMusicBee 3.6で少なくとも以下を検証する。

- field名
- comparison operator
- string/numeric semantics
- AND / OR nesting
- escaping
- null / empty
- multi-value field
- library domain

### TrackSort

MusicBee側でsort可能なprimitiveと、SDK側sortが必要な範囲を調査する。
性能特性をPublic docsで明示する。

### Timeline

- Position / Duration primitiveの単位
- current trackなしの戻り値
- Seek成功/失敗の戻り値
- stream等でDuration不明を区別できるか

### Lyrics

- missing lyricsとpresent-but-emptyをAPI上区別できるか
- lyrics typeをread可能か
- write時にlyrics typeを指定可能か
- removalの正式primitiveがあるか、empty writeのみか

### Artwork

- MusicBee 3.6でWebP artworkを直接保持・表示・取得可能か
- same ArtworkTypeに複数画像を保持できるか
- index APIの意味
- write/remove primitive
- supported image format
- practical artwork size distribution

### Metadata

正式にtyped fieldとして公開するfieldごとに以下を検証する。

- read representation
- writableか
- null / empty
- min/max/range
- multi-value delimiter semantics
- commit requirement

### Packaging

- MusicBee-loaded DLLのplatform target
- AnyCPU / x86 / x64挙動
- embedded dependency approach
- Costura.Fody等の互換性
- vendored official interface sourceのlicense / redistribution条件

## Research discipline

調査で判明する事項をユーザーの好みとして質問しない。

Public ContractをMusicBee実挙動から決める必要がある場合は、まず公式source / docs / forum / minimal verification Plugin等で確認し、結果をこの文書へ記録した上でDecisionへ反映する。
