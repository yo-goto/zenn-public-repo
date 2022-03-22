---
title: "Next.jsでChakraUIを使用しようとしたら useSystemColorMode のエラーが出たので解決"
emoji: "🥷"
type: "tech"
topics: [nextjs, chakraui, javascript, react]
published: true
date: 2022-03-22
url: "https://zenn.dev/estra/articles/nextjs-chakraui-error-usesysmtecolormode"
aliases: [記事_Next.jsでChakraUIを使用しようとしたら useSystemColorMode のエラーが出たので解決]
tags: [" #chakraUI #Framework/React/Next #type/zenn  "]
---

# 問題
https://chakra-ui.com/guides/getting-started/nextjs-guide

上記の公式ドキュメントを元に Next.js で ChakraUI を使用しようとして、プロバイダーのセットアップを行った後に `npm run dev` したら以下のエラーがでたので解決した。

```sh
TypeError: Cannot read properties of undefined (reading 'useSystemColorMode')
```

# 解決策
冒頭で紹介した Next.js で Chakara UI を使用するための [公式ドキュメント](https://chakra-ui.com/guides/getting-started/nextjs-guide)では以下の項目が記載されていたが、"Provider Setup" の項目だけ行ってしまうと冒頭の TypeErorr が出力されてしまう。

- Provider Setup
- **Customizing theme**
- Color Mode Script
- Notes on TypeScript
- ChakraProvider Props

結論としては、"**Customizing theme**" の項目を行うとエラーがでなくなる。最初読んだ限りでは、この項目はオプショナルのものかと思ったが、実はやらないとエラーがでるものだった。

また、Chakra UI の公式ドキュメントだけでなく、Next.js が提供している以下のサンプルプロジェクトをみると理解するのに役立った。

https://github.com/vercel/next.js/tree/2e530ee2992ae9e873945b288973367870577a45/examples/with-chakra-ui

このサンプルプロジェクトは以下のコマンドでローカルインストールできる。

```sh
npx create-next-app --example with-chakra-ui with-chakra-ui-app
# or
yarn create next-app --example with-chakra-ui with-chakra-ui-app
```

## 準備(プロジェクトの作成)
プロジェクトの作成は以下の通り。

[Create Next App](https://nextjs.org/docs/api-reference/create-next-app) を npx で使用してプロジェクトに Next.js のテンプレートを展開する。テスト用として最小限にしたいので [Next.js のチュートリアル](https://nextjs.org/learn/basics/create-nextjs-app/setup)で使用する example のテンプレートプロジェクトを使用する。

```sh
npx create-next-app nextjs-blog --use-npm --example "https://github.com/vercel/next-learn/tree/master/basics/learn-starter"
```

公式ドキュメントの [Installation](https://chakra-ui.com/guides/getting-started/nextjs-guide) に従って Chakra UI をインストールする。

```sh
npm i @chakra-ui/react @emotion/react@^11 @emotion/styled@^11 framer-motion@^6
```

これによって、各ライブラリのインストールバージョンは以下となる。

```json:dependencies
"dependencies": {
  "@chakra-ui/react": "^1.8.6",
  "@emotion/react": "^11.8.2",
  "@emotion/styled": "^11.8.1",
  "framer-motion": "^6.2.8",
  "next": "latest",
  "react": "17.0.2",
  "react-dom": "17.0.2"
}
```

ディレクトリ構造については、`src/pages/` ではなく Next.js のチュートリアル通りのものとして、テンプレートプロジェクトとして展開された `pages/` を使用する。

## Provider setup
[公式ドキュメント](https://chakra-ui.com/guides/getting-started/nextjs-guide)の "Provider Setup" の項目通りに、次のコードを `pages/_app.js` ファイルに追加。`ChakraProvider` で `Component` をラップして Chakra UI を使用できるように準備する。

````js:pages/_app.js
import { ChakraProvider } from '@chakra-ui/react'

function MyApp({ Component, pageProps }) {
    return (
        <ChakraProvider>
            <Component {...pageProps} />
        </ChakraProvider>
    )
}

export default MyApp
````

ただし、この状態で `npm run dev` すると冒頭の TypeErorr が起きる。

:::message
ちなみに、`MyApp` コンポーネントに渡す props は以下のものとなる。
-   `Component` prop はアクティブな `page` を示しており、ルート間で遷移するたびに `Component` は新しい `page` に変わる。従って、`Component` に渡されるあらゆる props はそのアクティブな `page` によって取得される。
-   `pageProps` は Next.js におけるデータ取得メソッドの１つによってプレロードされたページの初期 props を含むオブジェクト。初期 props がなければ空のオブジェクトになる。

参照: [Advanced Features: Custom App](https://nextjs.org/docs/advanced-features/custom-app)
:::

## Customizing theme
Provider Setup した状態のコードから最小限でエラーがでないように `pages/_app.js` ファイルに "Customizing theme" の項目に記載されているコードを少し修正して追記する。

```diff jsx:pages/_app.js
import { ChakraProvider } from '@chakra-ui/react'
+// 1. extendTheme 関数をインポート
+import { extendTheme } from '@chakra-ui/react';
+// 2. custom colorやfontなどで theme を拡張する
+const colors = {
+  brand: {
+    900: '#1a365d',
+    800: '#153e75',
+    700: '#2a69ac',
+  },
+}
+const theme = extendTheme({ colors })
+// 3. ChakraProvider に `theme` prop を渡す
function MyApp({ Component, pageProps }) {
    return (
-        <ChakraProvider>
+        <ChakraProvider theme={theme}>
            <Component {...pageProps} />
        </ChakraProvider>
    )
}

export default MyApp
```

これで、`npm run dev` をおこなっても冒頭の TypeError が出力されずにコンパイルを正しく実行できるようになった。

```sh
$ npm run dev
# エラーが出力されないようになった
```

Next.js で Chakra UI を使用しようとすると、このようにテーマのカスタマイズを行う必要があるようだ。ChakarUI ではデフォルトテーマとカスタマイズスタイリングを深く統合できる `extendThems` という関数を提供しており、この関数の返り値を `theme` に代入し、それを  `ChakraProvider` の prop として渡してあげる必要がある。

`extendTheme` 関数について具体的な使い方については次の公式ドキュメントに記載されている。

https://chakra-ui.com/docs/styled-system/theming/customize-theme#theme-extension-withdefaultsize

実は Chakra UI の公式ドキュメントだと、`App` と `MyApp` というように表記がゆれており、そもそもどこのファイルに記載すればよいのかわかりづらくなっていた。

`App` と `MyApp` については、次の Next.js のドキュメントを参照。

https://nextjs.org/docs/advanced-features/custom-app

`pages/_app.js` や `src/pages/_app.js` ファイルにおいて、各ページの初期化を行っている `App` コンポーネントを上書きできる。`App` コンポーネントを `_app.js` で上書きすることで主に以下のことができる。

- ページが変化する間もレイアウトを保持させる
- ページ遷移時に stata を保持する
- `componentDidCatch` を使用してエラーハンドリングをカスタマイズする
- page に追加のデータを挿入する
- グローバル CSS を追加する

チュートリアルやこのドキュメントでは、`App` なのか `MyApp` なのか表記ゆれしているが名前はなんでも良いように思われる(テストしてみたが名前は何でも良かった)。

Next.js のドキュメントについて、そこで紹介されている次のコードを見る限りでは `getInitialProps` メソッドを使用しない限りは、`App` を import する必要はないので、もし import するなら上書きするコンポーネントの名前は `App` ではなく `MyApp` などの名前にする必要がある、という話だと思われる。

```jsx:上記ドキュメントに記載されている_app.js
// import App from 'next/app'

function MyApp({ Component, pageProps }) {
  return <Component {...pageProps} />
}

// Only uncomment this method if you have blocking data requirements for
// every single page in your application. This disables the ability to
// perform automatic static optimization, causing every page in your app to
// be server-side rendered.
//
// MyApp.getInitialProps = async (appContext) => {
//   // calls page's `getInitialProps` and fills `appProps.pageProps`
//   const appProps = await App.getInitialProps(appContext);
//
//   return { ...appProps }
// }

export default MyApp
```

Next.js が提供しているサンプルプロジェクト `with-chakra-ui-app` では、`const theme = extendTheme({ colors })` で使用した `theme` を `pages/_app.js` ファイルに記載せず、`theme.js` というテーマ専用に分離したファイルへ記載して `export default` していた。実際に開発でテーマ設定のカスタマイズを行うさいには、そのように分離する方が良いだろうと思われる。

ちなみに、サンプルプロジェクトのディレクトリ構造は次のようになっていた。

```sh
.
├── node_modules/
├── package-lock.json
├── package.json
├── README.md
└── src/
    ├── components/
    │  ├── Container.js
    │  ├── CTA.js
    │  ├── DarkModeSwitch.js
    │  ├── Footer.js
    │  ├── Hero.js
    │  └── Main.js
    ├── pages/
    │  ├── _app.js
    │  ├── _document.js
    │  └── index.js
    └── theme.js
```

このサンプルプロジェクトでは、`theme.js` で `export default` した `theme` を `_app.js` ファイルにて `import theme from '../theme'` でインポートしているので、冒頭のエラーが起きないようになっている。

`src/pages` ディレクトリと `pages` ディレクトリの違いについては公式ドキュメントの以下のページを参照。
https://nextjs.org/docs/advanced-features/src-directory

ちなみに `src` を使用したディレクトリでのプロジェクト構造についてよさそうなものが以下の記事で記載されていた。

https://dev.to/vadorequest/a-2021-guide-about-structuring-your-next-js-project-in-a-flexible-and-efficient-way-472

## Color Mode Script
"Customizing theme" の項目についてはわかったが、公式ドキュメントの次の項目 "Color Mode Script" では、このサンプルプロジェクトのように、`theme` を分離して import する形をとっていたのでこれについて行おうとすると混乱の原因になっていた。

https://chakra-ui.com/guides/getting-started/nextjs-guide#color-mode-script

冒頭の TypeError についての解決は終わっているが、ローカルストレージとの正しい同期を機能させるためにやらないといけないという旨がかかれていたので、一応コードを調整してから追加する。

まず、`pages` ディレクトリに `_document.js` ファイルを作成し、公式ドキュメント通りのコードを追記。

```js:pages/_document.js
// pages/_document.js

import { ColorModeScript } from '@chakra-ui/react'
import NextDocument, { Html, Head, Main, NextScript } from 'next/document'
import theme from './theme'

export default class Document extends NextDocument {
  render() {
    return (
      <Html lang='en'>
        <Head />
        <body>
          {/* 👇 Here's the script */}
          <ColorModeScript initialColorMode={theme.config.initialColorMode} />
          <Main />
          <NextScript />
        </body>
      </Html>
    )
  }
}
```

`Document` と `pages/_document.js` ファイルの詳細については次のドキュメントを参照。

https://nextjs.org/docs/advanced-features/custom-document

ただし、このままだと `theme` は `pages/_app.js` で宣言してあるので、Next.js のサンプルプロジェクトのように `theme` にまつわる部分を `theme.js` としてファイルで分離させる必要がある。従って、`pages/_app.js` を以下のように変更する。

```diff jsx:pages/_app.js
import { ChakraProvider } from '@chakra-ui/react';
+import theme from '../styles/theme';
-// 1. extendTheme 関数をインポート
-import { extendTheme } from '@chakra-ui/react';
-// 2. custom colorやfontなどで theme を拡張する
-const colors = {
-  brand: {
-    900: '#1a365d',
-    800: '#153e75',
-    700: '#2a69ac',
-  },
-}
-const theme = extendTheme({ colors })
// 3. ChakraProvider に `theme` prop を渡す
function MyApp({ Component, pageProps }) {
    return (
        <ChakraProvider theme={theme}>
            <Component {...pageProps} />
        </ChakraProvider>
    );
}

export default MyApp
```

同様に `pages/_document.js` での import 先を変更する。

```diff js:pages/_document.js
// pages/_document.js

import { ColorModeScript } from '@chakra-ui/react'
import NextDocument, { Html, Head, Main, NextScript } from 'next/document'
-import theme from './theme'
+import theme from '../styles/theme'

export default class Document extends NextDocument {
  render() {
    return (
      <Html lang='en'>
        <Head />
        <body>
          {/* 👇 Here's the script */}
          <ColorModeScript initialColorMode={theme.config.initialColorMode} />
          <Main />
          <NextScript />
        </body>
      </Html>
    )
  }
}
```

ちなみにディレクトリ構造は Next.js のチュートリアル通りに以下のようなものとする(テンプレートプロジェクト通りの最小限構成)。

```sh
.
├── node_modules/
├── package-lock.json
├── package.json
├── README.md
├── pages/
│  ├── _app.js
│  ├── _document.js
│  └── index.js
├── public/
└── styles/
   └── theme.js
```

`styles` ディレクトリに `theme.js` ファイルを作成し、"Customizing theme" の項目で追加したコードを追加する。更に、外部で使用出来るように `export default` を行う。

```js:styles/theme.js
import { extendTheme } from '@chakra-ui/react'

const colors = {
  brand: {
    900: '#1a365d',
    800: '#153e75',
    700: '#2a69ac',
  },
}
const theme = extendTheme({ colors })

export default theme
```

先程、`pages/_document.js` にて `initialColorMode={theme.config.initialColorMode}` というコードを記載した。公式ドキュメントでは、デフォルトで `light` ということになっているが、一応明記しておき、さらに冒頭の TypeError にでてきた `useSystemColorMode` の設定について記載しておく。

https://chakra-ui.com/docs/styled-system/theming/theme#config

冒頭のエラーの内容を思い出すと、そもそも「未定義のプロパティを読み込めないよ」という内容だった。

```sh
TypeError: Cannot read properties of undefined (reading 'useSystemColorMode')
```

従って、`MyApp` に prop として渡すことのできる `theme` さえあれば、デフォルト値 `false` がよみこまれてエラーが回避できる仕様になっているようだ。今回は明示的に `true` にする(実は、`<ChakraProvider theme={theme}>` としなくてもただインポートするだけでローカルのエラーは回避できた)。

再び `styles/theme.js` を以下のように変更する。

```diff js:styles/theme.js
import { extendTheme } from '@chakra-ui/react'

const colors = {
  brand: {
    900: '#1a365d',
    800: '#153e75',
    700: '#2a69ac',
  },
}
+const config = {
+  useSystemColorMode: true,
+  initialColorMode: 'light',
+}
-const theme = extendTheme({ colors })
+const theme = extendTheme({ 
+  colors,
+  config,
+})

export default theme
```

これで OS の preference に合わせて light/dark のテーマが変わるようになった。

## ChakraProvider Props
`pages/_app.js` ファイルで使用する `ChakraProvider` に渡せる props について "ChakraProvider Props" の項目でリスト化されていた。

- `resetCSS` : 自動的に `<CSSRest />` を含ませる
    - 型: `boolean`
    - デフォルト値: `true`
- `theme` : オプショナルなカスタムテーマを使用する
    - 型: `Theme`
    - デフォルト値: `@chakra-ui/theme`
- `colorModeManager` : コンポーネント内でユーザーのカラーモード設定を永続化するマネージャー
    - 型: `StrongManager`
    - デフォルト値: `localStorageManager`
- `portalZIndex` : Portal 用の共通の z-index
    - 型: `number`
    - デフォルト値: `undefined`

`theme` にデフォルト値があるらしいので `theme` 単体で使おうとしたらやはりエラーがでた。`import theme from '@chakra-ui/theme'` してもダメだったので今までのやり方をやはり行う必要がある。


# Issue

検索したらこのエラーについての issue を見つけた。

https://github.com/vercel/next.js/issues/18941

