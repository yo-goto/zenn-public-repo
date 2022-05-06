---
title: "async/await[作成中]"
----

# Async function

Prosmise チェーンが分かったことでようやく async/await に入ることができます。非同期処理は Promise が主役です。async/await を使ってできることは Promise による非同期処理の利便性を高めることです。

:::message
asynchronous の発音記号は `/eɪsɪ́ŋkr(ə)nəs/` ですので、`async` の読み方は日本語だと"エイシンク"になるはずですが、まあアシンクでもエイシンクでもどっちでも良いでしょう。
:::

非同期関数(Async function)は**どんな時も必ず Promise インスタンスを返す関数**です。関数定義には `async` キーワードを使用します。そして、Promise インスタンスを返すわけですから、Promise チェーンが可能です。

```js:通常の関数定義
// returnStringInAsync.js
async function alwaysReturnPromise() {
  return "👾 Promise でラップされる文字列";
}

alwaysReturnPromise()
  .then((value) => console.log("🍓 履行値:", value))
  .catch((reason) => console.error("🥦 拒否理由:", reason))
  .finally(() => console.log("👍 チェーン最後に実行"));
```

ちゃんと Promise チェーンが出来ていますね。

```sh:実行結果
❯ deno run returnStringInAsync.js
🍓 履行値: 👾 Promise でラップされる文字列
👍 チェーン最後に実行
```

アロー関数だとこんな感じになります。

```js:アロー関数での定義
// returnNothingInAsync
const alwaysReturnPromise = async () => {
  // 何もしない
};

alwaysReturnPromise()
  .then((value) => console.log("🍓 履行値:", value))
  .catch((reason) => console.error("🥦 拒否理由:", reason))
  .finally(() => console.log("👍 チェーン最後に実行"));
```

Async function 内部で何も `return` しなくても、必ず Promsie インスタンスが返ってきます。この場合、返ってくる Promise インスタンスは履行値が `undefined` の履行状態になります。

```sh:実行結果
❯ deno run returnStringInAsync.js
🍓 履行値: 👾 Promise でラップされる文字列
👍 チェーン最後に実行
```

# await 式
Async function が便利なのは `await` 式を使って非同期処理などを待つことができるからです。この「待つ」ですが、非常に混乱させるワードなので注意してください。

例えば、お馴染みの非同期 API `fetch()` は Promise インスタンスを返してきましたね。`fetch()` は引数に渡した URL からデータを取得してくれます。ですが、ネットワーキングでリクエストを投げてレスポンスを受け取るまでには時間がかかります。通常は、こんな時間のかかる処理をメインスレッドで行っていたら無駄な待ち時間が発生してしまいますね。というわけで、データ取得は API を介してその作業を委任された環境(environment)がバックグラウンドで行ってくれます。

:::message
環境と非同期 API についての話は『非同期 API と環境』のチャプターで解説したので、詳しくはそちらを参照してください。
:::

```js
const GOOGLE = "https://www.google.com";
const fetchData = (url) => {
  return fetch(url)
    .then((response) => {
      if (!response.ok) {
        throw new Error("Error");
      }
      console.log(`[3] 👦 MICRO: got data from "${GOOGLE}"`);
      return response.text();
    });
};

console.log("[1] 🦖 MAINLINE: Start");

fetchData(GOOGLE)
  .then((data) => {
    console.log("[4] 👦 MICRO: 取得データ", data);
  })
  .catch((error) => {
    console.error(error.message);
  });

console.log("[2] 🦖 MAINLINE: End");
```

async/await は Promise チェーンはシュガーシンタックスであると言われます。実際にはそれ以上のものですが。

上記の Promise チェーンは async/await で書き直すことができます。

```js
const GOOGLE = "https://www.google.com";
const fetchData = async (url) => {
  try {
    const data = await fetch(url)
    .then((response) => {
        if (!response.ok) {
          throw new Error("Error");
        }
        console.log(`[3] 👦 MICRO: got data from "${GOOGLE}"`);
        return response.text();
      });
  } catch (error) {

  }
};

console.log("[1] 🦖 MAINLINE: Start");

fetchData(GOOGLE)
  .then((data) => {
    console.log("[4] 👦 MICRO: 取得データ", data);
  })
  .catch((error) => {
    console.error(error.message);
  });

console.log("[2] 🦖 MAINLINE: End");
```

# Promise チェーンだ

# V8 の await 式

V8 エンジンのブログにおいて async/await の内側が語られています。

https://v8.dev/blog/fast-async#await-under-the-hood

```js
async function foo(v) {
  const w = await v;
  return w;
}
```