---
title: "Next.js アプリに Storybook を導入する"
emoji: "🐕"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["nextjs", "storybook"]
published: true
---

## はじめに

この記事では Next.js アプリに Stobybook を導入する方法について書きます。
CSS Modules を使用してスタイリングを行う既存の Next.js アプリに導入することを前提としており、主に [Get started with Storybook and Next.js](https://storybook.js.org/blog/get-started-with-storybook-and-next-js/) に従って進めていきます。
Sass、Chakra UI などでスタイリングを行っている場合は別に設定が必要になります。別途[こちら](https://storybook.js.org/docs/react/configure/styling-and-css)を参照してください。

この記事が他の人の参考になれば幸いです。
また、この記事の内容に間違った記載がありましたら、指摘してもらえるとありがたいです。

## 環境

| 名前           | バージョン |
| -------------- | ---------- |
| macOS Monterey | 12.0.1     |
| Node.js        | 16.13.0    |
| React          | 17.0.2     |
| Next.js        | 12.0.7     |
| Storybook      | 6.4.9      |

## Storybook のセットアップ

以下のコマンドを実行し、Storybook と関連パッケージをインストールします。
パフォーマンスと Next.js との一貫性のためビルダーとして Webpack 5 を使用します。
コマンド実行中に `eslintPlugin` をインストールし、設定するか聞かれます。この自動設定は `.eslintrc` の拡張子が `json` の場合は対応していないので、本記事では `no` にし、後で設定します。

```bash:terminal
npx sb init --builder webpack5
```

上記のコマンド実行後、`npm run storybook` コマンドで Storybook を起動できます。

## Storybook 環境へのグローバル CSS の適用

Storybook 環境でもグローバル CSS が適用されるために `.storybook/preview.js` に以下のようにグローバル CSS のインポート文を追加します。

```js:.storybook/preview.js
import "../styles/globals.css";
```

## Storybook で next/image を扱うための設定の追加

`next/image` コンポーネントは Next.js によって自動的に最適化処理が行われます。
しかし、Storybook は Next.js とは独立した環境を構築するため、そのままではうまく表示されません。
そのため Storybook で next/image コンポーネントを適切に表示する設定を行います。

まず、Next.js で画像を配置する `public` ディレクトリを Storybook で扱うために `.storybook/main.js` に以下の設定を追加します。

```js:.storybook/main.js
module.exports = {
  ...
  staticDirs: ["../public"],
};

```

そして、Storybook では最適化されていない next/image コンポーネントを使用するために `.storybook/preview.js` に以下の内容を追加します。

```js:.storybook/preview.js
import * as NextImage from "next/image";

...

const OriginalNextImage = NextImage.default;

Object.defineProperty(NextImage, 'default', {
  configurable: true,
  value: (props) => <OriginalNextImage {...props} unoptimized />,
});
```

## モジュールパスエイリアスを対応づける

相対パスでインポートを行っている場合にこの設定は必要ないので読み飛ばしてください。

ベース URL、モジュールパスエイリアスを使用する場合は Storybook に対応関係を伝えないといけません。
`.storybook/main.js` に以下のように追加します。
この記事では `src/components`、`src/lib`、`src/models`、`src/pages`、`src/styles` ディレクトリの対応関係を設定しています。

```js:.storybook/main.js
const path = require("path");

module.exports = {
  ...
  webpackFinal: async (config) => {
    return {
      ...config,
      resolve: {
        ...config.resolve,
        alias: {
          ...config.resolve.alias,
          "@/components": path.resolve(__dirname, "../src/components"),
          "@/lib": path.resolve(__dirname, "../src/lib"),
          "@/models": path.resolve(__dirname, "../src/models"),
          "@/pages": path.resolve(__dirname, "../src/pages"),
          "@/styles": path.resolve(__dirname, "../src/styles"),
        },
      },
    };
  },
};
```

## Storybook に関する ESLint ルールの追加

手動で Storybook の ESLint プラグイン `eslint-plugin-storybook` をインストールし、適用します。
このプラグインは `.stories.*` もしくは `.story.*` の拡張子のファイルに対して Storybook のベストプラクティスルールを追加します。
以下のコマンドでインストールします。

```bash
npm i -D eslint-plugin-storybook
```

プラブインを適用するために `.eslint.json` を変更します。以下のように `extends` に `"plugin:storybook/recommended"` を追加してください。

```json
{
  "extends": ["plugin:storybook/recommended", "next/core-web-vitals"]
}
```

## 参考

- [Get started with Storybook and Next.js](https://storybook.js.org/blog/get-started-with-storybook-and-next-js/)
- [eslint-plugin-storybook](https://github.com/storybookjs/eslint-plugin-storybook)
