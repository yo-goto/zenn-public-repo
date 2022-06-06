---
title: "EPAsync book 管理ノート"
type: []
---
# 管理用ノート

タスク
- [x] `config.yml` ファイルでチャプター管理した方が良さそう
  - 内容を後で差し替えたり、追加したりする時に管理しづらい。
- [ ] yaml データを各データに記載する

追加内容
- [x] レンダリングパイプライン
- [x] タスク詳細
- [x] 擬似コード node など
- [x] 実行コンテキスト
- [x] Promise の状態の詳細
- [x] async/await
- [x] Top-level await

## チャプター管理

```yaml
chapters:
  - 1-epasync-begin # はじめに
  - f-epasync-asyncronous-apis # 非同期APIと環境
  - 2-epasync-event-loop # イベントループの概要と注意点
  - d-epasync-task-microtask-queues # タスクキューとマイクロタスクキュー
  - e-epasync-v8-engine # V8エンジン
  - b-epasync-callstack-execution-context #コールスタックと実行コンテキスト
  - c-epasync-what-event-loop # それぞれのイベントループ
  - a-epasync-promise-basic-consept # Promise の基本概念
  - 3-epasync-promise-constructor-executor-func # Promise コンストラクタと Executor 関数
  - 4-epasync-callback-is-sync-or-async # コールバック関数の同期実行と非同期実行
  - g-epasync-resolve-reject # resolve 関数と reject 関数の使い方
  - 5-epasync-multiple-promises # 複数の Promise を走らせる
  - 6-epasync-then-always-return-new-promise # then メソッドは常に新しい Promise を返す
  - 7-epasync-pass-value-to-the-next-chain # Promise チェーンで値を繋ぐ
  - 8-epasync-return-promise-in-then-callback # then メソッドのコールバックで Promise インスタンスを返す
  - 9-epasync-dont-next-promise-chain # Promise チェーンはネストさせない
  - 10-epasync-dont-use-side-effect # then メソッドのコールバックで非同期処理
  - 11-epasync-omit-return-by-arrow-shortcut # アロー関数で return を省略する
  - h-epasync-catch-finally # catch メソッドと finally メソッド
  - 12-epasync-wrapping-macrotask # 古い非同期APIをPromiseでラップする
  - 13-epasync-loop-is-nested # イベントループは内部にネストしたループがある
  - 14-epasync-chain-to-async-await # Promise チェーンから非同期関数へ
  - 15-epasync-v8-converting # V8 エンジンによる async/await の内部変換
  - 16-epasync-top-level-async # Top-level await
  - x-epasync-epilogue # エピローグ
```

