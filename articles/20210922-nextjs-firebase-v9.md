---
title: "Firebase SDK v9, Recoil で Next.js アプリの Google ログインを実装する"
emoji: "📑"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["typescript", "firebase", "nextjs", "recoil"]
published: true
---

## はじめに

この記事では TypeScript で書かれた Next アプリに Firebase JavaScript SDK v9 を利用して Google ログインを実装する方法について書きます. 認証するユーザは Recoil で管理します. Context API で管理する方法は[こちらの記事](https://zenn.dev/minguu42/articles/20210717-nextjs-typescript-auth.md)で書いています.
また, 前述した記事に載っていますので, この記事では Firebase プロジェクトの作成, Google プロバイダの有効化などの詳しい手順は省略させて頂きます.

この記事が他の人の参考になれば幸いです.
また, この記事の内容に間違った記載がありましたら, 指摘してもらえるとありがたいです.

## 環境

| 名前                    | バージョン |
| ----------------------- | ---------- |
| macOS Big Sur           | 11.6       |
| Node.js                 | 16.9.0     |
| TypeScript              | 4.4.3      |
| React                   | 17.0.2     |
| Next.js                 | 11.1.2     |
| Recoil                  | 0.4.1      |
| Firebase JavaScript SDK | 9.0.2      |

## 事前準備

Next アプリに Google ログインを実装するコードを書いていく前に Firebase プロジェクト等を作る必要があります. 主に以下のことを事前に行う必要があります.

- Firebase プロジェクトの作成
- Authentication, Google プロバイダの有効化
- Web アプリの追加とその設定情報のメモ

詳しくは[こちらの記事](https://zenn.dev/minguu42/articles/20210717-nextjs-typescript-auth.md)で行なっていますので, 必要に応じて参照してください.

## 環境変数ファイルを作成する

Firebase の設定情報はコード上に記述し, バージョン管理したくないので環境変数ファイル `.env.local` を作成し, 事前準備で得た設定情報を記述しておきます.

```bash:.env.local
NEXT_PUBLIC_FIREBASE_API_KEY=<apiKey>
NEXT_PUBLIC_FIREBASE_AUTH_DOMAIN=<authDomain>
NEXT_PUBLIC_FIREBASE_PROJECT_ID=<projectId>
NEXT_PUBLIC_FIREBASE_STORAGE_BUCKET=<storageBucket>
NEXT_PUBLIC_FIREBASE_MESSAGING_SENDER_ID=<messagingSenderId>
NEXT_PUBLIC_FIREBASE_APP_ID=<appId>
```

## firebase ライブラリのインストールとセットアップ

firebase ライブラリをインストールし, そのセットアップを行います.

```bash:terminal
npm i firebase
```

インストール後, 任意のファイル（この記事では `lib/firebase.ts`）で firebase ライブラリのセットアップを行います.
環境変数ファイル `.env.local` から Firebase の設定情報を読み込み, firebase ライブラリの初期化をしています.
また, `app` は他のファイルで使用するのでエクスポートしています.

```typescript:lib/firebase.ts
import { initializeApp } from "firebase/app";

const firebaseConfig = {
  apiKey: process.env.NEXT_PUBLIC_FIREBASE_API_KEY,
  authDomain: process.env.NEXT_PUBLIC_FIREBASE_AUTH_DOMAIN,
  projectId: process.env.NEXT_PUBLIC_FIREBASE_PROJECT_ID,
  storageBucket: process.env.NEXT_PUBLIC_FIREBASE_STORAGE_BUCKET,
  messagingSenderId: process.env.NEXT_PUBLIC_FIREBASE_MESSAGING_SENDER_ID,
  appId: process.env.NEXT_PUBLIC_FIREBASE_APP_ID,
};

export const app = initializeApp(firebaseConfig);
```

参考：[Add Firebase to your JavaScript project](https://firebase.google.com/docs/web/setup)

## ログイン, ログアウトを行う関数を定義する

任意のファイル（この記事では `lib/auth.ts`）にログイン, ログアウトを行う関数を定義します. `lib/auth.ts` に以下のように記述しました.

```tsx:lib/auth.ts
import { getAuth, signInWithRedirect, signOut, GoogleAuthProvider } from "firebase/auth";

import { app } from "./firebase";

export const login = (): Promise<void> => {
  const provider = new GoogleAuthProvider();
  const auth = getAuth(app);
  return signInWithRedirect(auth, provider);
};

export const logout = (): Promise<void> => {
  const auth = getAuth(app);
  return signOut(auth);
};
```

`signInWithRedirect()` で Google ログイン用のページにリダイレクトし, Google ログインを行います. また, provider, auth のプロパティを変更することで言語などを変更, 設定できます. 詳しくは [Authenticate Using Google Sign-In with JavaScript](https://firebase.google.com/docs/auth/web/google-signin) など参照してください.
`signOut()` でログアウトします.

## ユーザ認証を管理する関数を定義する

Recoil でユーザ認証を管理します. `lib/auth.ts` に以下のように追記しました.

```tsx:lib/auth.ts
import { useEffect, useState } from "react";
import { atom, useRecoilValue, useSetRecoilState } from "recoil";
import {
  User,
  getAuth,
  signInWithRedirect,
  signOut,
  onAuthStateChanged,
  GoogleAuthProvider,
} from "firebase/auth";

import { app } from "./firebase";

type UserState = User | null;

const userState = atom<UserState>({
  key: "userState",
  default: null,
  dangerouslyAllowMutability: true,
});

export const login = (): Promise<void> => {
  const provider = new GoogleAuthProvider();
  const auth = getAuth(app);
  return signInWithRedirect(auth, provider);
};

export const logout = (): Promise<void> => {
  const auth = getAuth(app);
  return signOut(auth);
};

export const useAuth = (): boolean => {
  const [isLoading, setIsLoading] = useState(true);
  const setUser = useSetRecoilState(userState);

  useEffect(() => {
    const auth = getAuth(app);
    return onAuthStateChanged(auth, (user) => {
      setUser(user);
      setIsLoading(false);
    });
  }, [setUser]);

  return isLoading;
};

export const useUser = (): UserState => {
  return useRecoilValue(userState);
};
```

`userState` は認証するユーザを保持する Recoil の状態です. ユーザがログインしている場合は `User`, ログインしていない場合は `null` になります. `useUser()` はその userState を他のコンポーネントで呼び出すための関数です.
`useAuth()` がユーザ認証を監視するための関数です. `onAuthStateChanged()` メソッドはユーザ認証を監視し, 変更があったときに引数のコールバック関数を実行します. その関数内でユーザの状態を更新しています.
`isLoading` は `onAuthStateChanged()` を実行中か確認するための状態です. この状態の値が true の時は `onAuthStateChanged()` でユーザを認証中です.

## ログイン, ログアウト, ユーザ認証の管理を実装する

上記で定義した関数を使用し, 実際にユーザ認証を実装します.
まず, `pages/_app.tsx` に以下のように追記しました.

```tsx:pages/_app.tsx
import "../styles/globals.css";
import type { AppProps } from "next/app";
import { RecoilRoot } from "recoil";

import { useAuth } from "../lib/auth";

type Props = {
  children: JSX.Element;
};

const Auth = ({ children }: Props): JSX.Element => {
  const isLoading = useAuth();

  return isLoading ? <p>Loading...</p> : children;
};

function MyApp({ Component, pageProps }: AppProps) {
  return (
    <RecoilRoot>
      <Auth>
        <Component {...pageProps} />
      </Auth>
    </RecoilRoot>
  );
}

export default MyApp;
```

`useAuth()` は `<RecoilRoot>` の子孫コンポーネントでしか使用できないので `<Auth>` コンポーネントを作成し, その中で使用しています. ユーザ認証中は`<p>Loading...</p>`のみが表示されます.

ログイン, ログアウトは以下のように任意の場所で関数を呼び出し, 行ないます.
また, `useUser()` も以下のように使用し, ユーザを取得できます.

```tsx:pages/index.tsx
...
import { useUser, login, logout } from "../lib/auth";

const Home: NextPage = () => {
  const user = useUser();

  const handleLogin = (): void => {
    login().catch((error) => console.error(error));
  };

  const handleLogout = (): void => {
    logout().catch((error) => console.error(error));
  };

  return (
    <div className={styles.container}>
      <Head>
        <title>Auth Example</title>
      </Head>

      <div>
        <h1>Auth Example</h1>
        {user !== null ? (
          <h2>ログインしている</h2>
        ) : (
          <h2>ログインしていない</h2>
        )}
        <button onClick={handleLogin}>ログイン</button>
        <button onClick={handleLogout}>ログアウト</button>
      </div>
    </div>
  );
};
...
```

## 参考

- [Add Firebase to your JavaScript project](https://firebase.google.com/docs/web/setup)
- [Authenticate Using Google Sign-In with JavaScript](https://firebase.google.com/docs/auth/web/google-signin)
- [Get Started with Firebase Authentication on Websites](https://firebase.google.com/docs/auth/web/start)
- [Recoil で Firebase Auth を使う](https://zenn.dev/yiwa/articles/3d4b91fd4fb467)
