# MusicBee.PluginSdk Design Documentation

このorphan `docs` branchは、MusicBee.PluginSdk自体の設計・調査・実装進捗を管理するための内部設計ドキュメントである。

Plugin利用者向けドキュメントはmain branchへ置く。

## 最初に読む文書

- [`current-state.md`](current-state.md)
  - 現在の設計全体、確定事項、未調査事項、次の開始位置
  - 新しいChatGPT/Codex sessionでは最初にこれを読む
- [`decisions/design-decisions.md`](decisions/design-decisions.md)
  - dig設計Q1〜Q190のDecision一覧
- [`research/musicbee-api.md`](research/musicbee-api.md)
  - MusicBee公式APIについて確認済みの事実と、Public Contract確定前に必要な追加調査

## 運用

設計を進める際は次を維持する。

1. 新しいDecisionを`decisions/design-decisions.md`へ追記する
2. 設計全体へ影響する場合は`current-state.md`も更新する
3. 調査で確定したMusicBee側の事実は`research/musicbee-api.md`へ反映する
4. 実装開始後は実装状況・未完了事項も`current-state.md`へ反映する

`current-state.md`だけを読んでも作業再開でき、詳細な理由が必要な場合にDecision / Research文書へ辿れる状態を維持する。

## ブランチ分離の理由

main branchの文書はPlugin実装者とAIがSDKを利用するための情報へ集中させる。

SDK自身の設計履歴・判断経緯・未確定調査・開発進捗を混在させると利用者向け情報のsignal-to-noise ratioが低下するため、これらはorphan `docs` branchで管理する。
