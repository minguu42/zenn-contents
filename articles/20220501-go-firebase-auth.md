---
title: "バックエンドのGoアプリでFirebase Authenticationのユーザを特定する"
emoji: "🙌"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["go", "firebase"]
published: true
---

## はじめに

アプリによって、フロントエンドとバックエンドでそれぞれ独立したアプリケーションを作成し、1つのアプリを構成する場合があると思います。
この記事では、そういった構成のフロントエンドでFirebase Authenticationを利用し、ユーザ認証を行っている場合に、バックエンドのGoアプリケーションでも現在ログインしているユーザを特定する方法について書きます。

この記事が他の人の参考になれば幸いです。
また、この記事の内容に間違った記載がありましたら、指摘してもらえるとありがたいです。

## 環境

| 名前                  | バージョン |
| --------------------- | ---------- |
| macOS Monterey        | 12.2.1     |
| Go                    | 1.17.6     |
| Firebase Admin Go SDK | 4.6.0      |

## 概要

バックエンドサーバで現在ログインしているユーザを特定するには**Firebase Admin SDK**を使用します。Admin SDKを使用することで、独自サーバとFirebase Authenticationを統合したり、ユーザや認証トークンを管理できます。
バックエンドサーバでユーザを特定する大まかな流れは以下のようになります。

1. ユーザがログインしているクライアントアプリケーションでIDトークンを取得し、HTTPSでバックエンドサーバに対して、そのIDトークンを含めたリクエストを送信する。
2. サーバでリクエストに含まれるIDトークンを確認し、ユーザを特定するクレーム（uid、ログインに使用したIDプロバイダなど）を抽出する。
3. そのクレームを使用して、ユーザを特定する

## 実装手順

### サービスアカウントの作成

Firebase Admin SDKを利用するためには、サービスアカウントでAdmin SDKを初期化する必要があります。そのため、サービスアカウントを作成し、サービスアカウントファイルを取得します。サービスアカウントファイルは認証情報を含むJSONファイルです。

[Firebaseコンソール](https://console.firebase.google.com/)で、[プロジェクトの設定] > [サービスアカウント]を開きます。そして、Firebase Admin SDKの[新しい秘密鍵の生成]をクリックし、[キーを生成]をクリックして、作成します。作成されたサービスアカウントファイルは公開せずに安全に保管します。

### Admin SDKの初期化

バックエンドのGoアプリケーションでAdmin SDKの初期化を行います。
まず、以下のコマンドでSDKをインストールします。

```bash
go get firebase.google.com/go/v4
```

そして、サービスアカウントファイルを使用してAdmin SDKを初期化します。初期化する方法は以下の3つです。

1. 環境変数`GOOGLE_APPLICATION_CREDENTIALS`を利用する方法
2. サービスアカウントファイルのパスをコード内に明示的に書く方法
3. 環境変数にJSONファイルを埋め込む方法

#### 1. 環境変数`GOOGLE_APPLICATION_CREDENTIALS`を利用する方法

この方法は公式ドキュメントで推奨されている方法です。環境変数`GOOGLE_APPLICATION_CREDENTIALS`にサービスアカウントファイルのファイルパスを設定します。

```bash
export GOOGLE_APPLICATION_CREDENTIALS="/home/user/Downloads/service-account-file.json"
```

そして、Goアプリケーションでは以下のように初期化します。

```go
app, err := firebase.NewApp(context.Background(), nil)
if err != nil {
	// エラー処理
}
```

#### 2. サービスアカウントファイルへのパスをコード内に明示的に書く方法

この方法では環境変数を指定する代わりにソースコードにサービスアカウントファイルのファイルパスを埋め込みます。

```go
opt := option.WithCredentialsFile("path/to/service-account-file.json")
app, err := firebase.NewApp(context.Background(), nil, opt)
if err != nil {
	// エラー処理
}
```

#### 3. 環境変数にJSONファイルを埋め込む方法

この方法はjqコマンドなどを利用して、JSONファイルを1行の文字列に変換し、その文字列を環境変数に設定します。そして、アプリケーションでは環境変数からJSONファイルの内容を読み取り、初期化を行います。
以下の例では環境変数`GOOGLE_CREDENTIALS_JSON`にJSONファイルの中身を設定しています。

```go
opt := option.WithCredentialsJSON([]byte(os.Getenv("GOOGLE_CREDENTIALS_JSON")))
app, err := firebase.NewApp(context.Background(), nil, opt)
if err != nil {
	// エラー処理
}
```

参考: [サーバーにFirebase Admin SDKを追加する](https://firebase.google.com/docs/admin/setup)

### IDトークンの確認

初期化後は`VerifyIDToken`メソッドでIDトークンを確認します。

```go
client, err := app.Auth(ctx)
if err != nil {
  	// エラー処理
}

token, err := client.VerifyIDToken(ctx, idToken)
if err != nil {
  // エラー処理
}

log.Printf("User ID: %v\n", token.UID)
```

`VerifyIDToken`メソッドの戻り値`token`は[`auth.Token`型](https://pkg.go.dev/firebase.google.com/go/v4/auth@v4.6.0#Token)の値です。
ユーザのIDであるUIDフィールドなどを持っており、ユーザの特定はこのUIDフィールドの値を利用します。

## 具体的な実装例

Goで書かれたREST APIでの実装例を紹介します。

`firebase.go`ではAdmin SDKの初期化と使用する関数を定義しています。
インタフェース`firebaseAppInterface`はテスト時に`firebaseApp`をモックするために使用します。

```go:firebase.go
// 省略
type firebaseAppInterface interface {
	VerifyIDToken(ctx context.Context, idToken string) (*auth.Token, error)
}

type firebaseApp struct {
	*firebase.App
}

func InitFirebaseApp() (*firebaseApp, error) {
	app, err := firebase.NewApp(context.Background(), nil, option.WithCredentialsJSON([]byte(os.Getenv("GOOGLE_CREDENTIALS_JSON"))))
    if err != nil {
        // エラー処理
    }
	return &firebaseApp{app}, nil
}

func (app *firebaseApp) VerifyIDToken(ctx context.Context, idToken string) (*auth.Token, error) {
	client, err := app.Auth(ctx)
    if err != nil {
        // エラー処理
    }
	token, err := client.VerifyIDToken(ctx, idToken)
    if err != nil {
        // エラー処理
    }
	return token, nil
}

// 省略
```

次の`auth.go`は実際にユーザの特定を行う処理を行うミドルウェアを定義します。
URL`/health`に対するエンドポイントはユーザの特定を必要としないので、アーリーリターンしています。

```go:auth.go
// 省略
type userKey struct{}

func Auth(db dbInterface, firebaseApp firebaseAppInterface) func(handler http.Handler) http.Handler {
	return func(next http.Handler) http.Handler {
		return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
			if r.URL.Path == "/health" {
				next.ServeHTTP(w, r)
				return
			}

			ctx := r.Context()

			if !strings.HasPrefix(r.Header.Get("Authorization"), "Bearer ") {
				// Authorization ヘッダのフォーマットが適切でない
			}
			idToken := strings.Split(r.Header.Get("Authorization"), " ")[1]
			token, err := firebaseApp.VerifyIDToken(ctx, idToken)
			if err != nil {
				// エラー処理
			}

			user, err := db.GetUserByDigestUID(hash(token.UID))
			if err != nil {
				// エラー処理
			}

			next.ServeHTTP(w, r.WithContext(context.WithValue(ctx, userKey{}, user)))
		})
	}
}

func hash(token string) string {
	bytes := sha256.Sum256([]byte(token))
	return hex.EncodeToString(bytes[:])
}
```

上記のミドルウェア`Auth`で特定したユーザをハンドラで取得するコードは以下のようになります。

```go
// 省略
func postTasks(w http.ResponseWriter, r *http.Requst) {
    user := r.Context().Value(userKey{}).(*User)
    // 省略
}
```

## 参考

- [Admin Auth APIの概要](https://firebase.google.com/docs/auth/admin)
- [IDトークンを検証する](https://firebase.google.com/docs/auth/admin/verify-id-tokens)
- [認証トークン](https://firebase.google.com/docs/auth/users#auth_tokens)
