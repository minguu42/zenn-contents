---
title: "OpenAPIからコードの自動生成を行うGoフレームワークogenを触ってみた"
emoji: "🦔"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["go", "openapi", "ogen"]
published: true
published_at: 2023-07-21 19:00
---

## はじめに

これまでGoでREST APIを開発する際のライブラリとして[chi](https://github.com/go-chi/chi)を使用していました。しかし、chiでREST APIの開発を行なっているとOpenAPIドキュメントで定義している仕様と実装との乖離が起きやすいという問題があります。そのため、OpenAPIドキュメントからAPIサーバのコードを生成する使いやすいGoフレームワークを探していました。そのとき見つけたのが[ogen](https://github.com/ogen-go/ogen)です。
この記事では、ogenの特徴と簡単な使い方を紹介したいと思います。

この記事が他の人の参考になったら幸いです。
また、この記事の内容に誤った記載がありましたら、指摘してもらえるとありがたいです。

## ogenとは

改めて、ogenとは[OpenAPI Specification v3](https://swagger.io/specification/v3/)で書かれたOpenAPIドキュメントから、HTTPサーバやHTTPクライアントのGoコードを自動生成するフレームワークです。
ogenを使用することで開発者はリクエストハンドラの実装のみに集中できます。

ogenの特徴は、[公式ドキュメント](https://ogen.dev)では次のように示されています。

- リフレクションなし
  - 生成されるJSONエンコーティングは最適化され、`encoding/json`の制限を回避し、パフォーマンスを改善するために、[jx](https://github.com/go-faster/jx)を使用する
  - バリデーションは仕様から生成される
- ボイラープレートなし
  - 構造体はOpenAPI Specification v3から生成される
  - 引数、ヘッダ、URLクエリは仕様に沿って構造体にパースされる
  - `uuid`、`data`、`date-time`、`uri`などの文字列フォーマットはGoの型で直接表現される
  - OneOfでは識別器または暗黙的な型推論によってSum型が生成される
  - OptionalとNullableは可能であればポインタなしでサポートされる
- OpenTelemetryと互換性のあるトレースとメトリクスをサポートする

## ogenで単純なAPIサーバを実装する

ここからは実際のコード例を用いてogenを用いて単純なAPIサーバを開発する方法を説明します。

### コードを生成する

まず、以下のコマンドで`ogen`をインストールします。

```bash
go install -v github.com/ogen-go/ogen/cmd/ogen@latest
```

次にOpenAPIドキュメントを用意します。
この記事では次のOpenAPIドキュメント`openapi.yaml`を使用します。

```yaml:openapi.yaml
openapi: 3.0.3
info:
  version: 1.0.0
  title: Note API
paths:
  /notes:
    post:
      summary: メモを作成する
      operationId: createNote
      requestBody:
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/Note'
        required: true
      responses:
        200:
          description: 成功
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/Note'
  /notes/{noteID}:
    get:
      summary: メモを取得する
      operationId: getNoteByID
      parameters:
        - name: noteID
          in: path
          required: true
          schema:
            type: integer
            format: int64
      responses:
        200:
          description: 成功
          content:
            application/json:
              schema:
                $ref: "#/components/schemas/Note"
        404:
          description: メモがありません
          content:
            application/json:
              schema:
                type: object
                properties:
                  code:
                    type: integer
                  message:
                    type: string
                required:
                  - code
                  - message
components:
  schemas:
    Note:
      type: object
      properties:
        id:
          type: integer
          format: int64
        title:
          type: string
        content:
          type: string
      required:
        - id
        - title
        - content
```

このOpenAPIドキュメントからogenのコードを生成するには以下のコマンドを実行します。
`-package`フラグは生成されたコードのパッケージ名、`-target`フラグは生成されるコードのディレクトリを指定します。`-clean`フラグは既に生成したコードを削除して、新しくコードを生成することを示します。

```bash
ogen -package ogen -target ogen -clean ./openapi.yaml
```

このコマンドの実行後、次のように`ogen`ディレクトリにコードが生成されたと思います。

```
.
├── ogen
│   ├── oas_cfg_gen.go
│   ├── oas_client_gen.go
│   ├── oas_handlers_gen.go
│   ├── oas_interfaces_gen.go
│   ├── oas_json_gen.go
│   ├── oas_middleware_gen.go
│   ├── oas_parameters_gen.go
│   ├── oas_request_decoders_gen.go
│   ├── oas_request_encoders_gen.go
│   ├── oas_response_decoders_gen.go
│   ├── oas_response_encoders_gen.go
│   ├── oas_router_gen.go
│   ├── oas_schemas_gen.go
│   ├── oas_server_gen.go
│   └── oas_unimplemented_gen.go
```

### 生成したコードのHandlerインタフェースを実装する構造体を定義する

そして、APIサーバを実装するためには`oas_server_gen.go`の`Handler`インタフェースを満たす構造体を実装します。

```go:oas_server_gen.go
// Handler handles operations described by OpenAPI v3 specification.
type Handler interface {
	// CreateNote implements createNote operation.
	//
	// メモを作成する.
	//
	// POST /notes
	CreateNote(ctx context.Context, req *Note) (CreateNoteRes, error)
	// ListNote implements listNote operation.
	//
	// メモを取得する.
	//
	// GET /notes/{noteID}
	ListNote(ctx context.Context, params ListNoteParams) (*Note, error)
}
```

以下の内容で`main.go`を作成し、次のように`Handler`インタフェースを満たす`noteService`構造体を実装しました。

```go:main.go
type noteService struct {
	notes map[int64]ogen.Note
	mux   sync.Mutex
}

func (n *noteService) CreateNote(_ context.Context, req *ogen.Note) (*ogen.Note, error) {
	n.mux.Lock()
	defer n.mux.Unlock()

	n.notes[req.ID] = *req
	return req, nil
}

func (n *noteService) GetNoteByID(_ context.Context, params ogen.GetNoteByIDParams) (ogen.GetNoteByIDRes, error) {
	n.mux.Lock()
	defer n.mux.Unlock()

	note, ok := n.notes[params.NoteID]
	if !ok {
		return &ogen.GetNoteByIDNotFound{
			Code:    1,
			Message: "メモが見つかりません",
		}, nil
	}
	return &note, nil
}
```

ここで注目して欲しいのはリクエストボディが`CreateNote`の引数`req`に、パラメータが`GetNoteByID`の引数`params`にと、OpenAPIドキュメントに記載した定義がしっかりと構造体にマッピングされていることです。さらに、レスポンスに関しても`GetNoteByID`メソッドを見ると、正常時のレスポンスは`*ogen.Note`に、エラーレスポンスは`*ogen.GetNoteByIDNotFound`にマッピングされています。 このようにogenではリクエストやレスポンスを表す構造体を定義する際にリフレクションを使用していないので、OpenAPIドキュメントの仕様と実装の乖離を発生させず、リクエストとレスポンスの値を安全に扱えます。

また、各リクエストハンドラメソッドの2つ目の返り値`error`は全て`nil`となっています。
この2つ目の返り値の`error`は[Convenient errors](https://ogen.dev/docs/concepts/convenient_errors)という機能で使うものであり、ここでは使用していないためです。 この機能はとても便利なので、興味があればドキュメントを読んで使ってみてください。

### サーバを起動する

`Handler`インタフェースを満たす構造体を定義できたら、サーバを実行する処理を追加します。
`main.go`に次のコードを追記しました。

```go:main.go
func main() {
	service := &noteService{
		notes: map[int64]ogen.Note{},
	}
	s, err := ogen.NewServer(service)
	if err != nil {
		log.Fatalln(err)
	}
	if err := http.ListenAndServe(":8080", s); err != nil {
		log.Fatalln(err)
	}
}
```

追記したコードでは`Handler`インタフェースを満たす構造体を`ogen.NewServe`関数の第1引数に渡し、サーバを初期化しています。`ogen.NewServer`関数の返り値`s`は`net/http`パッケージの`http.Handler`インタフェースを満たすので、ミドルウェアは通常の`net/http`パッケージの場合と同様の方法で利用できます。

これでAPIサーバの実装が完了しました。
次のコマンドを実行して、動作を確認してみてください。

```bash
go run .
```

```bash
$ curl -X POST --location "http://localhost:8080/notes" \
    -H "Accept: application/json" \
    -H "Content-Type: application/json" \
    -d "{
          \"id\": 1,
          \"title\": \"foo\",
          \"content\": \"bar\"
        }"
{"id":1,"title":"foo","content":"bar"}
```

```bash
$ curl -X GET --location "http://localhost:8080/notes/1" \
    -H "Accept: application/json"
{"id":1,"title":"foo","content":"bar"}
```

```bash
$ curl -X GET --location "http://localhost:8080/notes/1" \
    -H "Accept: application/json"
{"code":1,"message":"メモが見つかりません"}
```

正常時のレスポンスがしっかりと生成されているのはもちろんなのですが、エラー時のレスポンスもしっかりと生成されていることが確認できますね。

## さいごに

ogenを用いて単純なAPIサーバを実装する手順を紹介しましたが、ogenの使用感が伝わったでしょうか？ ogenはこの記事で紹介したようにコードを自動生成してボイラーコードを減らしてくれるだけでなく、OpenAPIドキュメントに記載された値の制約を実現するバリデーション機能があったり、OpenTelemetryに対応していたりするなど嬉しい点が他にもあります。
他のGoフレームワークに比べて使用者がまだ少ないので、気になったらぜひ使ってみてください。

## 参考

- [ogen](https://github.com/ogen-go/ogen)
- [ogenドキュメント](https://ogen.dev)
