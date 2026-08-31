# Scope Reset Decisions — Q210-Q228

最終更新: 2026-08-31

この文書は、初期の「広範な汎用SDKを先に設計する」方針から、実PluginでdogfoodするSlim Core方針へ変更したDecisionを記録する。
`design-decisions.md` の過去Decisionと矛盾する場合、Q番号が新しい本書のDecisionを優先する。

## Q210 — MusicBee callback形状をsemantic eventへ正規化

- `ReceiveNotification`とspecial well-known callbackの違いを通常のPlugin application codeへ漏らさない
- SDK Feature側でsemantic eventへ正規化する
- ただし各special callbackは使用前にMusicBee 3.6で実挙動を調査する

## Q211 — Slim formal Coreへscope縮小

完全な第三者向けMusicBee SDKを先に作り切らない。
自分の複数Pluginで再利用できる薄い正式Coreを作り、Featureは実需ベースで追加する。

## Q212 — Feature APIはdogfood時に追加

開発順序:
1. Core
2. Lyrics/Artwork PluginでIPC + Metadata/Lyrics/Artwork
3. Stream Deck PluginでPlayer/NowPlaying/Event
4. 実績を見て一般化

Playlist、Queue、Query、TrackEditSession等はDeferred。

## Q213 — IPCはPhase 1 Coreから外す

IPCの必要性自体は高いが、Lyrics/Artwork Plugin開始時に実要件を使って設計する。

## Q214 — Source Generatorをやめる

Roslyn Source Generatorではなく、Templateから通常の可視C# entrypoint sourceをPlugin Repositoryへ配置する。
AI/Codexからも読みやすく、debug/build complexityを減らす。

## Q215 — Special callbackは必要時のみ追加

標準entrypointへ全special callbackを生成しない。
method presence自体がMusicBeeへのfeature signalになる場合があるため、Featureが必要な場合だけ追加する。

## Q216 — Standard General Plugin callback

標準Templateには以下を含める。
- Initialise
- ReceiveNotification
- Close
- Configure
- SaveSettings
- Uninstall

## Q217 — Minimal WinForms settings UI helper

SDKはMusicBee `Configure(IntPtr)` とWinForms Panel hostingのglueを提供する。
Pluginは通常のWinForms Controlを実装する。
自動UI生成・大型binding frameworkは作らない。

## Q218 — Lightweight JSON settings store

- MusicBee persistent storage path + Plugin ID directory
- typed JSON load/save
- missing fileはdefault
- temp + replace/moveによる安全な書き込み

schema migration framework、migration backup、provider abstraction等はDeferred。

## Q219 — Small settings page contract

`IPluginSettingsPage<TSettings>` 相当の小さなcontractを設ける。
FrameworkがConfigure/SaveSettings lifecycleと永続化をbridgeし、PluginがUI/model mappingとvalidationを担当する。

## Q220 — Settings stateはediting modelに保持

WinForms Controlそのものをsettings stateの唯一のsource of truthにしない。
UIとは別にediting `TSettings` modelを持つ。

## Q221 — Configure再呼出時にediting modelを再利用

当初は同じediting modelを再利用する方針とした。

**Status: Superseded by Q222**

MusicBeeからPreferences Cancel/Closeを通知する明確なpublic callbackを確認できず、Cancel済み変更が次回Configureへ残る危険があるため撤回。

## Q222 — Configureごとにediting modelを再生成

`Configure()`を新しい編集境界として扱い、呼出ごとに永続化済みsettingsから新しいediting modelを生成する。
`SaveSettings()`でvalidationして保存する。
未保存modelは次回Configureへ持ち越さない。

## Q223 — SDK-owned PluginMetadata

Plugin applicationはMusicBee `PluginInfo` を直接構築しない。
SDK-owned `PluginMetadata` を定義し、FrameworkがInterop `PluginInfo`へ変換する。

## Q224 — VersionはAssembly metadataから取得

PluginMetadataへVersionを二重記述しない。
Project/Assembly metadataをsingle source of truthとする。
Release時はGit tagからVersionをbuild propertyへ渡す方式を採用する方向。

## Q225 — MusicBeeにはMajor.Minor.Patchのみ渡す

MusicBee `PluginInfo`はSemVer prereleaseを表現できないため、Major/Minor/PatchだけをVersionMajor/VersionMinor/Revisionへ渡す。
完全なSemVerはAssembly/NuGet/GitHub Release側で保持する。

## Q226 — Compatibility revisionはSDKが自動決定

Plugin作者はMinInterfaceVersion/MinApiRevisionを指定しない。
CoreとFeatureが必要条件を持ち、使用される要求の最大値をFrameworkがMusicBee PluginInfoへ設定する。

## Q227 — PluginIdは明示指定必須

Assembly名や表示名から生成しない。
Pluginの永続identityとしてPlugin作者が一度明示的に決める。
Persistent data path等の基点にする。

## Q228 — PluginId書式

lowercase reverse-domain形式とする。

- ASCIIのみ
- 許可文字: `a-z`, `0-9`, `.`, `-`
- 例: `jp.hitsujin.musicbee.lyricsartwork`
- Framework起動時にvalidation

## このDecision群でSupersede/Deferredになった主な旧方針

- Source Generator: Superseded
- global MusicBee dispatcher: audit Q203でSuperseded
- `%AppData%` default settings path: audit Q206でSuperseded
- Configure間で未保存settings modelを保持: Q222でSuperseded
- 広範なFeature API事前実装: Deferred
- IPC Phase 1実装: Deferred
- 高機能settings migration/provider framework: Deferred

## 次の検討

Phase 1 Coreで本当に実装前決定が必要な事項だけを続ける。
候補:
- PluginMetadataの残りのcontract
- Plugin bootstrap / DI registration API
- Uninstall時のpersistent data削除policy
- settings validation/error handling
- Start/Stop timeout policy

実装開始前に主要な未確定事項がなくなったことを確認し、ユーザーから設計合意を得る。
