---
title: "第１章 - API を提供する環境と実行メカニズム"
cssclass: zenn
date: 2022-06-16
modified: 2022-11-02
AutoNoteMover: disable
tags: [" #type/zenn/book  #JavaScript/async "]
alieases: [EPAsync 第１章]
---

# この章について

第１章では API を提供する環境と JavaScript の実行メカニズムについて見ていきます。この章の内容は「非同期処理」についてメタ的な視点で解説し、裏側の機構などについて学習します。

先取りで[第２章](sec-02-epasync)と[第３章](sec-03-epasync)の知識(Promise chain と async/await)を利用している場面があるので、この章が難しい場合には具体的なコードから解説している第２章の内容から読み進めるのも良いかもしれません。ただし、この章では非同期処理の実行順序を予測するために必要なイベントループの解説があり、**重要度としてはすべての章の中で最も高い**ので注意してください。

# 前準備

この章の内容を理解するために、JSConfEU で行われた Philip Roberts 氏の講演動画である『What the heck is the event loop anyway?』をはじめに見ておくことを強く推奨しています(この動画は非常に重要なのでこの本を読まずとも必ず見るようにしてください)。

https://www.youtube.com/watch?v=8aGhZQkoFbQ

この動画で、"イベントループ" と "コールスタック"、"非同期 API" の概要を掴んでからこの章の内容を読んでいただけると良いと思います。

# チャプター

- [非同期 API と環境](f-epasync-asynchronous-apis)
- [同期 API とブロッキング](f-epasync-synchronus-apis)
- [イベントループの概要と注意点](2-epasync-event-loop)
- [タスクキューとマイクロタスクキュー](d-epasync-task-microtask-queues)
- [V8 エンジンについて](e-epasync-v8-engine)
- [コールスタックと実行コンテキスト](b-epasync-callstack-execution-context)
- [それぞれのイベントループ](c-epasync-what-event-loop)
