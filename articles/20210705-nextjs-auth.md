---
title: "Next.jsでFirebase Authenticationを利用し、Googleログインを実装した"
emoji: "👻"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["nextjs", "firebase", "javascript"]
published: true
---

## はじめに

この記事では、Firebase Authenticationを使ってNextアプリにGoogleログインを実装する方法を記述します。

Nextアプリの初期化、Firebaseアカウントの作成は済んでいることを前提とします。

この記事が他の人の参考になれば幸いです。
また、この記事の内容に間違った記載がありましたら、指摘してもらえるとありがたいです。

## 環境

| 名前                    | バージョン |
| ----------------------- | ---------- |
| macOS Big Sur           | 11.4       |
| Node.js                 | 16.4.1     |
| Next.js                 | 11.0.1     |
| React                   | 17.0.2     |
| Firebase JavaScript SDK | 8.7.0      |

## Firebaseでプロジェクトを作成し、アプリを登録する

### Firebaseプロジェクトを作成する

まずブラウザで[Firebase console](https://console.firebase.google.com/?hl=ja)にアクセスし、Firebaseプロジェクトを作成します。
`プロジェクト名`、`Googleアナリティクスの設定`は任意です。

### AuthenticationとGoogleログインを有効にする

プロジェクトが作成できたら、作成したプロジェクトのページに飛び、左側のタブの[Authentication]を押します。
Authenticationのページに飛び、画面にある[始める]のボタンを押し、始めます。

Sign-in methodのログインプロバイダに複数のログイン方法が並んでいる画面が表示され、この記事ではGoogleを選択し、[有効にする]にチェックを入れ、有効にします。
`プロジェクトの公開名`と`プロジェクトのサポートメール`は適切に設定して保存します。

本番環境で使用する場合はSign-in methodの承認済みドメインからlocalhostのドメインを削除するなど利用するドメインを適切に設定します。

### アプリをFirebaseに登録する

プロジェクト概要のページに戻り、アプリを登録します。
画面の中央にあるiOS、Android、Webなどのアイコンの中からWebのアイコンを選択し、任意の`アプリのニックネーム`を入力し、アプリの登録を行います。

登録した後に[Firebase SDKの追加]に移り、画面にソースコードが表示されます。
`firebaseConfig`変数に代入されている値を後で使用するなので、コピーして保存して置きます。

## Nextアプリで実装する

### 環境変数のファイルを作成する

Firebaseの設定情報は公開したくないので`.env.local`ファイルを作成し、アプリからは環境変数として読み込みます。
先ほどコピーした`firebaseConfig`変数のプロパティ値をそれぞれ以下のように`.env.local`に記述します。

`NEXT_PUBLIC_`のプリフィックスはブラウザで動作するプログラムで環境変数を読み込む際に必要です。
Googleアナティクスの有効無効によって、環境変数の数が異なる可能性があります。

```al:.env.local
NEXT_PUBLIC_FIREBASE_API_KEY=<apiKey>
NEXT_PUBLIC_FIREBASE_AUTH_DOMAIN=<authDomain>
NEXT_PUBLIC_FIREBASE_PROJECT_ID=<projectId>
NEXT_PUBLIC_FIREBASE_STORAGE_BUCKET=<storageBucket>
NEXT_PUBLIC_FIREBASE_MESSAGING_SENDER_ID=<messagingSenderId>
NEXT_PUBLIC_FIREBASE_APP_ID=<appId>
```

### Firebaseライブラリのインストールとセットアップ

`npm i firebase`コマンドでFirebaseライブラリをインストールします。

インストール後に任意のファイル（この記事では`lib/firebase.js`）を作成し、Firebaseライブラリのセットアップを行います。

`lib/firebase.js`は以下のように記述します。
if文の処理はコンポーネントの再レンダリングで複数回`firebase`の初期化が行われないようにしています。

```js:lib/firebase.js
import firebase from 'firebase/app'
import 'firebase/auth'

const firebaseConfig = {
    apiKey: process.env.NEXT_PUBLIC_FIREBASE_API_KEY,
    authDomain: process.env.NEXT_PUBLIC_FIREBASE_AUTH_DOMAIN,
    projectId: process.env.NEXT_PUBLIC_FIREBASE_PROJECT_ID,
    storageBucket: process.env.NEXT_PUBLIC_FIREBASE_STORAGE_BUCKET,
    messagingSenderId: process.env.NEXT_PUBLIC_FIREBASE_MESSAGING_SENDER_ID,
    appId: process.env.NEXT_PUBLIC_FIREBASE_APP_ID,
}

if (firebase.apps.length === 0) {
    firebase.initializeApp(firebaseConfig)
}

export const auth = firebase.auth()
export default firebase
```

### ログイン処理の実装

Googleログインを実装します。

この記事では、`lib/AuthContext.js`に認証関係のデータをグローバルで扱うためのコンテキスト、`pages/_app.js`にコンテキストを読み込む処理、`pages/index.js`に実際のログイン、ログアウト、ログイン状態の確認の処理を記述しています。

`lib/AuthContext.js`ファイルは以下のように記述しました。
コンテキストコンポーネントを作成し、実際のログイン、ログアウト、ログイン状態の確認は`useAuth`を通して使用できるようにしています。

Googleログインで使用できるログイン方法はリダイレクトの他にポップアップを使用する方法もあります。

```js:lib/AuthContext.js
import { createContext, useContext, useEffect, useState } from 'react'
import firebase, { auth } from '../lib/firebase'

const AuthContext = createContext()

export const useAuth = () => {
    return useContext(AuthContext)
}

const AuthProvider = ({ children }) => {
    const [currentUser, setCurrentUser] = useState(null)
    const [loading, setLoading] = useState(true)

    const login = () => {
        const provider = new firebase.auth.GoogleAuthProvider()
        return auth.signInWithRedirect(provider)
    }

    const logout = () => {
        return auth.signOut()
    }

    useEffect(() => {
        return auth.onAuthStateChanged(user => {
            setCurrentUser(user)
            setLoading(false)
        })
    }, [])

    const value = {
        currentUser,
        login,
        logout
    }

    return (
        <AuthContext.Provider value={value}>
            {loading ? <p>loading...</p> : children}
        </AuthContext.Provider>
    )
}

export default AuthProvider
```

`pages/_app.js`ファイルは以下のように記述しました。
アプリ全体で`useAuth`で渡す値を使用したいので、`MyApp`コンポーネントの返すコンポーネントとして記述しています。

```js:pages/_app.js
省略
import AuthProvider from "../lib/AuthContext";

function MyApp ({ Component, pageProps }) {
  return (
    <AuthProvider>
      <Component {...pageProps} />
    </AuthProvider>
  )
}
省略
```

`pages/index.js`ファイルは以下のように記述しました。
`login`、`logout`関数のエラー処理はしていませんが、以下のように使用できます。
`currentUser`の値が`null`かオブジェクトかでログイン状態の確認ができます。

```js:index.js
import { useAuth } from "../lib/AuthContext";

export default function Home() {
  const { currentUser, login, logout } = useAuth()

  const handleLoginButton = () => {
    login()
  }

  const handleLogoutButton = () => {
    logout()
  }

  return (
    <div>
      <h1>Hello, next-auth</h1>
      {currentUser &&
      <div>
        <h2>ログインしています.</h2>
        <button onClick={handleLogoutButton}>ログアウト</button>
      </div>
      }
      {!currentUser &&
      <div>
        <h2>ログインしていません.</h2>
        <button onClick={handleLoginButton}>ログイン</button>
      </div>}
    </div>
  )
}
```

## 参考

- [FirebaseをJavaScriptプロジェクトに追加する](https://firebase.google.com/docs/web/setup?sdk_version=v8)
- [JavaScriptでGoogleログインを使用して認証する](https://firebase.google.com/docs/auth/web/google-signin)
- [React Authentication Crash Course With Firebase And Routing](https://www.youtube.com/watch?v=PKwu15ldZ7k)
