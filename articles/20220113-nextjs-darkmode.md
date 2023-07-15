---
title: "Next.js（with CSS Modules）アプリにダークモードを実装する"
emoji: "🎉"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["nextjs"]
published: true
---

## はじめに

この記事では CSS Modules でスタイリングを行なっている Next.js アプリにダークモードを実装する方法について書きます。この記事のコードは TypeScript で書いており、状態管理は Recoil で行っています。

この記事が他の人の参考になれば幸いです。
また、この記事の内容に間違った記載がありましたら、指摘してもらえるとありがたいです。

## 環境

| 名前           | バージョン |
| -------------- | ---------- |
| macOS Monterey | 12.0.1     |
| Node.js        | 16.13.0    |
| React          | 17.0.2     |
| Next.js        | 12.0.7     |
| Recoil         | 0.5.2      |

## 実装内容

この記事では以下の要件を満たすようなダークモードを実装します。

- Recoil でテーマを管理し、ボタンのクリックで簡単にテーマを切り替えられる。
- デフォルトでは `prefers-color-scheme` からユーザが OS などで設定しているテーマを取得し、適用する。
- ユーザがテーマを切り替えた場合はそのテーマを local storage で記憶し、local storage にテーマが保存されている場合はそのテーマを適用する。
- フラッシュ（最初にデフォルトのテーマを表示した後に適用するテーマに切り替わること）を起こさず、最初から適用するテーマで表示されるようにする。

実装方法としては JavaScript でテーマを扱う状態を持ち、CSS でテーマを識別するために HTML のルート要素 `html` に `data-theme` 属性を追加します。

そのため CSS Modules のスタイリングは以下のように行います。
CSS 変数を使用する場合は以下のようにグローバル CSS `styles/globals.css` でそれぞれの CSS 変数を宣言し、適用する箇所に挿入します。

```css:styles/globals.css
:root[data-theme="light"] {
  /* ライトテーマで適用するプロパティ値 */
  --c-primary: #6750a4;
  --c-on-primary: #625b71;
}

:root[data-theme="dark"] {
  /* ダークテーマで適用するプロパティ値 */
  --c-primary: #d0bcff;
  --c-on-primary: #ccc2dc;
}
```

```css
.container {
  /* CSS 変数の値を挿入する */
  color: var(--c-on-primary);
  background-color: var(--c-primary);
}
```

また、子孫セレクタを使用して以下のようにスタイリングできます。

```css
/* ライトテーマで適用 */
.container {
  color: red;
}

/* ダークテーマで適用 */
[data-theme="dark"] .container {
  color: darkred;
}
```

## テーマとテーマを管理する関数を定義する

まず、テーマの状態 `themeState` を定義し、その状態を扱う関数を定義します。
`useTheme` は React コンポーネントでテーマを扱うためのカスタムフックで、テーマ `theme` とテーマを切り替える関数 `toggleTheme` を返します。

```jsx:lib/theme.ts
import { atom, useRecoilState, useSetRecoilState } from "recoil";

export type Theme = "light" | "dark";

const themeState = atom<Theme>({
  key: "themeState",
  default: "light",
});

export const useSetTheme = () => useSetRecoilState(themeState);

export const useTheme = () => {
  const [theme, setTheme] = useRecoilState(themeState);

  const toggleTheme = () => {
    const newTheme = theme === "light" ? "dark" : "light";
    setTheme(newTheme);
    window.localStorage.setItem("theme", newTheme);
    const root = window.document.documentElement;
    root.setAttribute("data-theme", newTheme);
  };

  return { theme, toggleTheme };
};
```

## Theme コンポーネントを追加する

最初にページを読み込む時に `html` 要素に `data-theme` 属性を設定し、JavaScript で読み込む必要があります。
そのために `Theme.tsx` に以下のように `ThemeProvider` コンポーネントを定義します。

```jsx:Theme.tsx
import { useEffect } from "react";
import { Theme, useSetTheme } from "@/lib/theme";

type Props = {
  children: JSX.Element;
};

const ThemeProvider = ({ children }: Props): JSX.Element => {
  const setTheme = useSetTheme();

  useEffect(() => {
    const root = window.document.documentElement;
    const initialColorValue = root.getAttribute("data-theme");
    setTheme(initialColorValue as Theme);
  }, [setTheme]);

  return (
    <>
      <script
        dangerouslySetInnerHTML={{
          __html: `!function(){let e;const t=window.localStorage.getItem("theme");if(null!==t)e=t;else{e=window.matchMedia("(prefers-color-scheme: dark)").matches?"dark":"light"}document.documentElement.setAttribute("data-theme",e)}();`,
        }}
      />
      {children}
    </>
  );
};

export default ThemeProvider;
```

`ThemeProvider` コンポーネントの `useEffect` はマウント後に `data-theme` の値を読み込み、テーマをセットします。
`ThemeProvider` コンポーネントの `<script>` タグはフラッシュを防ぐために使用します。
`<script>` タグに記述した JavaScript スクリプトの実行後に `body` 要素のコンテンツのマウントされるので最初から正しいテーマを適用できます。
圧縮していない `<script>` タグの JavaScript スクリプトは以下のようになります。
ローカルストレージにテーマがあったら適用し、なかったら `prefers-color-scheme` からテーマを取得し、`html` 要素の `data-theme` 属性に設定します。

```jsx:圧縮していない <script> タグのスクリプト
(function () {
  let theme;
  const storageTheme = window.localStorage.getItem("theme");
  if (storageTheme !== null) {
    theme = storageTheme;
  } else {
    const mql = window.matchMedia("(prefers-color-scheme: dark)");
    theme = mql.matches ? "dark" : "light";
  }

  const root = document.documentElement;
  root.setAttribute("data-theme", theme);
})();
```

作成した `Theme` コンポーネントを `pages/_app.tsx` の `MyApp` コンポーネントの子要素に追加します。

```diff jsx:pages/_app.tsx
import type { AppProps } from "next/app";
import { RecoilRoot } from "recoil";

import Theme from "@/components/functional/Theme";
import "@/styles/globals.css";

const MyApp = ({ Component, pageProps }: AppProps): JSX.Element => {
  return (
    <RecoilRoot>
-     <Component {...pageProps} />
+     <Theme>
+       <Component {...pageProps} />
+     </Theme>
    </RecoilRoot>
  );
};

export default MyApp;
```

## ブラウザに現在のテーマを伝える

`input[type="date"]` 要素のカレンダーなどブラウザで表示される UI もダークモードに対応させるためにブラウザに現在のテーマを伝えます。

`_document.tsx` に以下のように `<meta>` タグを追加します。

```diff jsx:pages/_document.tsx
class MyDocument extends Document {
  ...

  render() {
    return (
      <Html lang="ja">
        <Head>
+         <meta name="color-scheme" content="light dark" />
        </Head>
        <body>
          <Main />
          <NextScript />
        </body>
      </Html>
    );
  }
}
```

そして、`styles/globals.css` に以下のように追記します。

```diff css:styles/globals.css
:root[data-theme="light"] {
+ color-scheme: light;
  ...
}

:root[data-theme="dark"] {
+ color-scheme: dark;
  ...
}
```

## 参考

- [The Quest for the Perfect Dark Mode](https://www.joshwcomeau.com/react/dark-mode/)
- [next-themes](https://github.com/pacocoursey/next-themes)
- [ダークモード対応へのいくつかのアプローチ](https://zenn.dev/ken7253/articles/darkmode-approach)
