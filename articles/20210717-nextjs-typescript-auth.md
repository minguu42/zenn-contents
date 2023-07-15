---
title: "Next.js（TypeScript）でFirebaseを利用し、Googleログインを実装する"
emoji: "🌟"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["typescript", "firebase", "nextjs"]
published: true
---

## はじめに

この記事では、Firebase Authenticationを使ってTypeScriptを使用したNextアプリにGoogleログインを実装する方法を記述します。
JavaScriptを使用したNextアプリにGoogleログインを実装する方法は[こちら](https://zenn.dev/minguu42/articles/20210705-nextjs-auth)に記述しています。

この記事が他の人の参考になれば幸いです。
また、この記事の内容に間違った記載がありましたら、指摘してもらえるとありがたいです。

## 環境

| 名前          | バージョン |
| ------------- | ---------- |
| macOS Big Sur | 11.4       |
| Node.js       | 16.4.1     |
| TypeScript    | 4.3.5      |
| React         | 17.0.2     |
| Next.js       | 11.0.1     |

## 適当なNextアプリの作成

以下のコマンドで`auth-example`というNextアプリを作成しました。
このアプリにGoogleログインを実装していきます。

```bash:terminal
npx create-next-app --ts --use-npm auth-example
```

## Firebaseの設定

以下の流れでFirebaseの設定を行っていきます。

1. Firebaseプロジェクトの作成
2. Authenticationの有効化
3. Googleプロバイダの有効化
4. （任意）承認済みドメインの設定
5. ウェブアプリの追加

[Firebase console](https://console.firebase.google.com/)にアクセスし、Firebaseプロジェクトを作成します。
適当な`プロジェクト名`を入力し、プロジェクトを作成します。
`Googleアナリティクス`の設定は任意です。

プロジェクトの作成後、左側のナビゲーションからAuthenticationのページに飛び、[始める]ボタンでAuthenticationを有効にします。

その後、AuthenticationのSign-in methodタブで`Google`プロバイダを有効にします。
`プロジェクトの公開名`、`プロジェクトのサポートメール`は適当なものを設定し、保存します。

また、Sign-in methodタブのページ下部で`承認済みドメイン`の設定ができます。
今回は変更しません。

左側のナビゲーションからプロジェクトの概要に飛び、`</>`のボタンでウェブアプリを追加します。
適当な`アプリのニックネーム`を入力し、アプリを登録してください。
その後、`Firebase SDKの追加`で表示されるソースコードは、Nextアプリで使用するので、コピーして保存しておきます。

## NextアプリでのGoogleログインの実装

以下の流れでNextアプリにGoogleログインを実装します。

1. 環境変数ファイル`.env.local`の作成
2. Firebaseライブラリのセットアップ
3. ログイン処理の実装

### 環境変数ファイル`.env.local`の作成

プロジェクトディレクトリに`.env.local`ファイルを作成し、環境変数を設定します。
先ほどコピーしたソースコードの`firebaseConfig`変数のプロパティ値を環境変数として読み込むために以下のように記述します。
`NEXT_PUBLIC_`のプリフィックスはクライアントで動作するNextアプリが環境変数を読み込むために必要です。
また、Googleアナティクスの有効無効によって、設定する環境変数の数が以下の場合と異なる可能性があります。

```text:.env.local
NEXT_PUBLIC_FIREBASE_API_KEY=<apiKey>
NEXT_PUBLIC_FIREBASE_AUTH_DOMAIN=<authDomain>
NEXT_PUBLIC_FIREBASE_PROJECT_ID=<projectId>
NEXT_PUBLIC_FIREBASE_STORAGE_BUCKET=<storageBucket>
NEXT_PUBLIC_FIREBASE_MESSAGING_SENDER_ID=<messagingSenderId>
NEXT_PUBLIC_FIREBASE_APP_ID=<appId>
```

### Firebaseライブラリのセットアップ

以下のコマンドでfirebaseライブラリをインストールします。

```bash:terminal
npm i firebase
```

インストール後、任意のファイル（この記事では`lib/firebase.ts`）でfirebaseライブラリのセットアップを行います。

`lib/firebase.ts`は以下のように記述しました。
環境変数からFirebaseの設定情報を読み込み、firebaseライブラリを初期化しています。
また、再レンダリングなどの際に、初期化が複数回行われないようにif文で条件分岐させています。

```typescript:lib/firebase.ts
import firebase from "firebase/app";
import "firebase/auth";

const firebaseConfig = {
    apiKey: process.env.NEXT_PUBLIC_FIREBASE_API_KEY,
    authDomain: process.env.NEXT_PUBLIC_FIREBASE_AUTH_DOMAIN,
    projectId: process.env.NEXT_PUBLIC_FIREBASE_PROJECT_ID,
    storageBucket: process.env.NEXT_PUBLIC_FIREBASE_STORAGE_BUCKET,
    messagingSenderId: process.env.NEXT_PUBLIC_FIREBASE_MESSAGING_SENDER_ID,
    appId: process.env.NEXT_PUBLIC_FIREBASE_APP_ID,
};

if (firebase.apps.length === 0) {
    firebase.initializeApp(firebaseConfig);
}

export const auth = firebase.auth();
export default firebase;
```

### ログイン処理の実装

実際のログイン処理を実装します。

この記事では、ユーザの認証関連の情報を`useContext`を使用し、グローバルで扱います。
そのため、`lib/AuthContext`に認証関連の情報を扱うコンテキストを、`pages/_app.js`にコンテキストを読み込む処理を、`pages/index.js`にログイン、ログアウト関数の呼び出しを記述します。

以下の流れで実装していきます。

1. `lib/AuthContext.tsx`で`AuthContext`の作成
2. `pages/_app.tsx`で`AuthContext`の読み込み
3. `pages/index.tsx`で`AuthContext`の使用

#### `lib/AuthContext.tsx`で`AuthContext`の作成

`lib/AuthContext.tsx`を作成し、以下のように記述しました。

`createContext`でユーザ、ログインやログアウトを扱う関数を保持する`AuthContext`を作成しています。
`AuthContext`はカスタムHook`useAuth`を通して使用できます。
`login`関数の`auth.signInWithRedirect`関数でリダイレクトでログインする方法を使用していますが、ポップアップを表示し、ログインする方法も存在します。

```tsx:lib/AuthContext.tsx
import { createContext, useState, useEffect, useContext } from "react";
import { User } from "@firebase/auth-types";

import firebase, { auth } from "../lib/firebase";

type AuthContextType = {
    currentUser: User | null;
    login?: () => Promise<void>;
    logout?: () => Promise<void>;
};

const AuthContext = createContext<AuthContextType>({ currentUser: null });

export const useAuth = () => {
    return useContext(AuthContext);
};

type Props = {
    children?: JSX.Element;
};

const AuthProvider = ({ children }: Props): JSX.Element => {
    const [currentUser, setCurrentUser] = useState<User | null>(null);
    const [isLoading, setIsLoading] = useState(true);

    const login = () => {
        const provider = new firebase.auth.GoogleAuthProvider();
        return auth.signInWithRedirect(provider);
    };

    const logout = () => {
        return auth.signOut();
    };

    useEffect(() => {
        return auth.onAuthStateChanged((user: User | null) => {
            setCurrentUser(user);
            setIsLoading(false);
        });
    }, []);

    const value: AuthContextType = {
        currentUser,
        login,
        logout,
    };

    return (
        <AuthContext.Provider value={value}>
            {isLoading ? <p>Loading...</p> : children}
        </AuthContext.Provider>
    );
};

export default AuthProvider;
```

#### `pages/_app.tsx`で`AuthContext`の読み込み

`pages/_app.tsx`を以下のように変更しました。

`<AuthProvider>`を`<Component>`の親コンポーネントにすることで全てのコンポーネントから`AuthContext`の情報にアクセスできるようにしています。

```tsx:pages/_app.tsx
import '../styles/globals.css'
import type { AppProps } from 'next/app'
import AuthProvider from "../lib/AuthContext";

function MyApp({ Component, pageProps }: AppProps) {
  return (
      <AuthProvider>
        <Component {...pageProps} />
      </AuthProvider>
  )
}

export default MyApp

```

#### `pages/index.tsx`で`AuthContext`の使用

`pages/index.tsx`を以下のように変更しました。

これは`useAuth`を使って`AuthContext`を利用する一つの例です。
`currentUser`の値でユーザがログインしているかどうかの判別ができます。

```tsx:pages/index.tsx
import styles from '../styles/Home.module.css'
import {useAuth} from "../lib/AuthContext";

export default function Home() {
const {currentUser, login, logout} = useAuth()

  return (
    <div className={styles.container}>
      <main className={styles.main}>
          {!currentUser && <button onClick={login}>ログイン</button>}
          {currentUser &&
          <div>
              <p>{currentUser.email} でログイン中</p>
              <button onClick={logout}>ログアウト</button>
          </div>}
      </main>
    </div>
  )
}
```
