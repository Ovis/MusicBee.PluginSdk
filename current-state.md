# MusicBee.PluginSdk — Current State

最終更新: 2026-08-31

この文書は、MusicBee.PluginSdk の設計検討を別チャット・別作業環境から継続するための正本である。
過去の広範なSDK設計は `decisions/design-decisions.md` および監査文書に残すが、現在はQ211以降のスコープ縮小を優先する。

## 目的

MusicBee 3.6系向けPluginを、自分の実Pluginで再利用できる「薄いが正式なSDK Core」として整備する。
完全な第三者向けMusicBee SDKを先に作り切ることはしない。Feature APIは実Pluginで必要になった時点で追加し、dogfoodしながら一般化する。

最初の実利用Pluginは別Plugin・別Repositoryとして2つ想定する。

1. Lyrics / Album Artwork更新Plugin
   - MusicBee側はGeneral Plugin
   - Lyrics更新後にArtwork更新などのアプリケーションロジックを持つ
   - Provider優先順位等の設定UIを持つ
   - HTTP / HTML解析 / 外部API / Cache / Rate Limit等はmodern .NET companion host側へ寄せる
2. Stream Deck操作Plugin
   - Play/Pause/Next/Previous等をMusicBeeへ送る
   - Playback state / Track情報 / Artwork等をStream Deck側へ返す
   - MusicBee側はnet48、外部側はmodern .NETまたはNode/TypeScript等を想定

## 現在の開発順序

1. MusicBee.PluginSdk Coreを完成させる
2. Lyrics / Artwork Plugin実装時にIPCとMetadata/Lyrics/Artwork APIを必要な分だけ追加する
3. Stream Deck Plugin実装時にPlayer/NowPlaying/Event APIを必要な分だけ追加する
4. 複数Pluginで実際に共通化価値が確認できたものだけ一般化する

Playlist、PlayingTracks/Queue、TrackQuery、汎用TrackEditSession等の広範なFeature API設計は現時点では保留する。

## Phase 1 Coreの対象

- vendored `MusicBeeInterface.cs` のInterop隔離
- MusicBeeロード側は.NET Framework 4.8
- Templateで生成する可視の `MusicBeePlugin.Plugin` partial class
- Framework bootstrap
- SDK-owned Plugin metadata
- MusicBee Interface Version / API Revision互換性検証
- `Microsoft.Extensions.DependencyInjection`
- `Microsoft.Extensions.Logging`
- lifecycle bridge
- MusicBee persistent storage path + stable Plugin ID
- 軽量typed JSON settings store
- 最小限のWinForms settings UI glue

Phase 1には含めないもの:

- IPC実装
- Source Generator
- Player / Metadata / Lyrics / Artwork等のFeature API wrapper
- 汎用event queue frameworkの作り込み
- Playlist / Queue / Query等の未使用Feature
- schema migration framework
- 自動設定UI生成 / property binding framework

## Entrypoint

Source Generatorは初期設計から外した。
`dotnet new`等のTemplateによって、Plugin Repositoryへ通常のC# sourceとして小さなentrypointを配置する。

MusicBeeの公式interfaceは `namespace MusicBeePlugin` の `public partial class Plugin` を前提としているため、Template側も同じpartial classとしてFrameworkへdelegateする。

Standard General Plugin templateには次のcallbackを含める。

- `Initialise`
- `ReceiveNotification`
- `Close`
- `Configure`
- `SaveSettings`
- `Uninstall`

`OnSelectedFilesChanged`、dockable panel等のspecial callbackは、Featureが必要になった場合だけTemplate/Plugin sourceへ追加する。
MusicBeeではmethod presence自体が機能シグナルになる場合があるため、special callbackを標準entrypointへ無条件生成しない。

## Lifecycle

MusicBee作者の説明・実API調査を踏まえ、global dispatcherは設けない。
通常の `ReceiveNotification` はPlugin用threadから呼ばれ、そのthreadからMusicBee APIを呼び出せる。UI thread affinityが必要なAPIだけ、そのFeature固有でmarshalする。

- `Initialise`: MusicBee API interface初期化、metadata/compatibility確認、DI等のFramework setup。長時間startup処理は行わない
- `PluginStartup`: Framework-managed `StartAsync` を開始してcallback自体は速やかにreturn。成功でRunning、timeout/exceptionでFaulted
- `Close(PluginCloseReason)`: `StopAsync`を開始しFramework timeoutまで同期的に完了を待つ。return後もcleanupが生存するとは仮定しない
- automatic restartは行わない

`ShutdownStarted` notificationには依存しない。

## Callback / Event方針

MusicBeeは通常notification (`ReceiveNotification`) とspecial well-known callback (`OnSelectedFilesChanged`等) を併用する。
SDKでFeature Eventを追加する場合は、このcallback形状の差をPlugin application codeへ漏らさずsemantic eventへ正規化する。
ただし、実際に利用する各callbackは抽象化前にMusicBee 3.6で挙動を確認する。

## Plugin metadata

Plugin applicationはMusicBeeの生 `PluginInfo` を直接構築せず、SDK-owned `PluginMetadata` を定義する。FrameworkがMusicBee `PluginInfo` へ変換する。

Plugin固有として明示する主な値:
- `PluginId`
- Name
- Description
- Author
- 必要なnotification設定
- settings UIに必要なConfigurationPanelHeight等

Framework側で決定する値:
- PluginInfoVersion
- MinInterfaceVersion
- MinApiRevision
- VersionMajor / VersionMinor / Revision

### Plugin version

Versionの正はproject/assembly metadataとする。PluginMetadataへVersionを二重記述しない。

Release workflowはBlogGeneratorで採用している方式を流用する方向:
- `v1.2.3` / `v1.2.3-beta.1` のようなGit tagでrelease
- GitHub ActionsでtagからSemVerを取得・検証
- build/packへ `-p:Version=<tag version>` を渡す
- FrameworkはPlugin assembly metadataからVersionを取得する

MusicBee `PluginInfo` は整数Major/Minor/Revisionしか持たないため、MusicBeeへはSemVerのMajor.Minor.Patchのみ渡す。
例: `1.2.3-beta.1` もMusicBee上は `1.2.3`。完全なSemVerはAssembly/NuGet/GitHub Release側で保持する。

### MusicBee compatibility

`MinInterfaceVersion` / `MinApiRevision` はPlugin作者に指定させない。
Coreおよび将来の各Featureが自身の必要revisionを持ち、有効な要求の最大値をFrameworkがPluginInfoへ設定する。

Publicな対応表現は「MusicBee 3.6 support」とし、技術的にはInterface Version + API Revisionで判定する。最低revisionは実際に使用するprimitiveから導出する。

## Plugin ID

`PluginId` は必須のstable identityであり、Assembly名、表示名、Repository名から自動生成しない。

書式:
- lowercase reverse-domain形式
- ASCIIのみ
- 許可文字は `a-z` / `0-9` / `.` / `-`
- 例: `jp.hitsujin.musicbee.lyricsartwork`
- Framework起動時にvalidationする

Plugin IDはMusicBee persistent storage配下のPlugin directory名に使用する。将来IPCを追加する場合もidentityの基点として利用できる。

## Persistent data / Settings

Plugin data rootは `%AppData%` 等をSDKが独自決定せず、MusicBee公式API `Setting_GetPersistentStoragePath()` を使用する。その直下にPlugin IDのdirectoryを作る。

SettingsはPhase 1では軽量JSON storeに限定する。
- typed JSON load/save
- file missing時はdefault settings
- temp fileを使った安全な書き込み + replace/move
- schema migration frameworkは作らない
- migration backup systemは作らない
- provider abstraction / in-memory provider等も必要になるまで作らない

## Settings UI

SDK CoreはMusicBee固有のWinForms hosting glueだけを提供する。自動UI生成や汎用binding frameworkは作らない。
Pluginは通常のWinForms `Control` とsettings model間のmapping/validationを実装する。SDK側には `IPluginSettingsPage<TSettings>` 相当の小さなcontractを設ける方向。

確認済み事項:
- `PluginInfo.ConfigurationPanelHeight > 0` でPreferences > Pluginsに設定領域が確保される
- `Configure(IntPtr panelHandle)` にMusicBeeが用意したPanelのhandleが渡される
- PluginはそのPanelへControlを追加する
- `SaveSettings()` はPreferencesのApply/Save時にMusicBeeから呼ばれ、Plugin自身が永続化する
- PreferencesのCancel/CloseをPluginへ通知する明確なpublic callbackは確認できない
- Configureの呼出回数やControl破棄タイミングを強く仮定しない

編集model lifecycle:
1. `Configure()`が呼ばれるたび、永続化済み `TSettings` から新しい編集modelを生成する
2. WinForms Controlはそのmodelを編集する
3. `SaveSettings()`でmodelをvalidationしてJSON Storeへ保存する
4. Controlそのものをsettings stateの唯一のsource of truthにしない
5. Cancelを検知できないため、未保存modelを次回Configureへ持ち越さない

以前の「Configure再呼出時に未保存modelを再利用する」案(Q221)はQ222によりSuperseded。

## IPC

IPCはPhase 1 Coreには含めない。
Lyrics / Artwork Pluginの実装開始時に実要件を使って設計・実装する。
MusicBee側がnet48である一方、HTTP/HTML/API等は最新.NETを利用したcompanion hostへ分離する基本方針は維持する。
Named Pipeは有力候補だが、過去に決めた詳細protocol/reconnect/capability等を初期Coreの実装要件とはしない。

## MusicBee API調査で確定している重要事項

- 現行公式C# interface: `https://getmusicbee.com/download/plugins/MusicBeeInterface.cs`
- `PluginInfoVersion = 1`
- `MinInterfaceVersion = 43`
- `MinApiRevision = 57`
- API Revision >= 58 はMusicBee v3.6へ対応
- Startup処理は `Initialise` ではなく `PluginStartup` が適切
- `ReceiveNotification` は通常Plugin threadから呼ばれ、そのthreadからMusicBee APIを呼べる
- `OnSelectedFilesChanged(string[] urls)` はGUI threadで呼ばれるspecial callback
- `PluginCloseReason`: MusicBeeClosing / UserDisabled / StopNoUnload
- `Close`は同期callback
- `Setting_GetPersistentStoragePath()` が存在する
- `Configure(IntPtr)` / `SaveSettings()` / `Uninstall()` はwell-known plugin callback
- `Library_GetLyrics`は存在するがdedicated `Library_SetLyrics`はない
- Artworkはindex-based `Library_GetArtworkEx` / `Library_SetArtworkEx`
- Player play state、position、volume、mute、shuffle(bool)、repeat等のprimitiveが存在する

Feature APIの詳細は、実Pluginで必要になった時点で `research/musicbee-api.md` と公式interfaceを再確認する。

## 過去Decisionの扱い

Q1-Q210には完全な汎用SDKを前提にした多数の設計Decisionがある。
Q211以降のscope resetによって、以下は「誤りとして削除」ではなく「必要性が実証されるまでDeferred」とする。

- 広範なApi/Framework package分割
- aggregate facade + 全Feature interface
- Source Generator
- 汎用DI event queue/coalescing framework
- Player/NowPlaying/Metadata/Lyrics/Artworkの事前全面設計
- Track Edit Session
- Playlist / Queue / Query / Selection等
- 高機能IPC protocol
- settings migration/provider framework
- Testing packageの広範なfake API

MusicBee実挙動調査によって誤りと判明した前提（global dispatcher、%AppData% default等）は監査Correctionを優先する。

## 次のチャットでの再開位置

Q228まで確定済み。

直近Decision:
- Q211: slim formal Coreへscope縮小
- Q212: Feature APIは実Plugin実装時にdogfood追加
- Q213: IPCはPhase 1から外す
- Q214: Source Generatorをやめ、Template-generated visible C# entrypoint
- Q215: special callbackは必要なFeatureだけ追加
- Q216: standard General Plugin templateは Initialise / ReceiveNotification / Close / Configure / SaveSettings / Uninstall
- Q217: minimal WinForms settings UI helper
- Q218: lightweight JSON settings store
- Q219: small settings page contract
- Q220: settings stateはControlではなくediting modelで保持
- Q221: Configure再呼出時にediting model再利用 → Q222でSuperseded
- Q222: Configureごとに永続化済みsettingsからediting modelを再生成
- Q223: SDK-owned PluginMetadata
- Q224: VersionはAssembly metadataから自動取得
- Q225: MusicBeeにはSemVer Major.Minor.Patchのみ渡す
- Q226: MinInterfaceVersion/MinApiRevisionはSDKが使用Core/Featureから自動決定
- Q227: PluginIdは明示指定必須
- Q228: PluginIdはlowercase reverse-domain形式、`a-z0-9.-`のみ

次はPhase 1 Coreの未確定事項を洗い出し、実装前に必要なDecisionだけdigする。
候補: `PluginMetadata`の残りのcontract、Plugin bootstrap/DI登録API、Uninstall時のdata削除policy、settings validation/error handling、Start/Stop timeoutの扱い。

まだ実装へ移行する合意は取っていない。新しいチャットでもdigを継続し、主要な未確定事項がなくなってからユーザーへ設計合意を確認する。
