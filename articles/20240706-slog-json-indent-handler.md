---
title: "Goのslogでインデント付きのJSONを出力するハンドラを実装する"
emoji: "🎃"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["go", "log", "logging", "slog"]
published: true
---

JSONの構造化ログを出力するのに`slog`パッケージはとても便利です。
しかし、`slog`パッケージで用意されているハンドラではインデントのないJSONしか出力できず、開発時にローカルでログを確認する際は少し不便です。
そのため、本記事では開発環境向けにインデント付きのJSONを出力するハンドラを実装する方法を紹介します。

## 実装

インデント付きのJSONを出力するハンドラの実装方法はいくつかあると思いますが、本記事では単純に`slog.JSONHandler`をラップする方法で実装します。
これは本番環境で使用することを想定する`slog.JSONHandler`との差異を可能な限り無くすためです。

他の実装方法として、レコードを`map[string]any`の形式で保持し、その値を`MarshalIndent`関数でJSONデータに変換して出力するという実装方法も考えられます。
しかし、こちらの方法だと出力されるJSONのフィールドの並び順がアルファベット順に変わってしまうため、本記事ではこの方法で実装するのを避けました。

`slog.Handler`インタフェースは次の4つのメソッドを持ちます。本記事では`JSONIndentHandler`構造体を定義し、この構造体が`slog.Handler`インタフェースを満たすように実装していきます。

```go
type Handler interface {
	Enabled(context.Context, Level) bool
	Handle(context.Context, Record) error
	WithAttrs(attrs []Attr) Handler
	WithGroup(name string) Handler
}
```

### JSONIndentHandler構造体

まず、`JSONIndentHandler`構造体を次のように宣言します。

```go
type JSONIndentHandler struct {
	handler slog.Handler
	w       io.Writer
	mu      *sync.Mutex
	buf     *bytes.Buffer
}
```

`handler`フィールドはラップするハンドラを、`w`フィールドはログの出力先を、`mu`フィールドはログの出力時に排他制御を行うためのミューテックスを保持します。
重要なのは`buf`フィールドです。本記事の実装では`slog.JSONHandler`が出力した内容をそのまま`w`に書き込まずに、このバッファで一時的に保持します。そして、バッファの内容をインデント付きのJSONデータに再度加工して`w`に書き込みます。

この`JSONIndentHandler`構造体を初期化する`NewJSONIndentHandler`関数は次のように宣言します。

```go
func NewJSONIndentHandler(w io.Writer, opts *slog.HandlerOptions) *JSONIndentHandler {
	buf := &bytes.Buffer{}
	return &JSONIndentHandler{
		handler: slog.NewJSONHandler(buf, opts),
		w:       w,
		mu:      &sync.Mutex{},
		buf:     buf,
	}
}
```

ここから個々のメソッドの実装に入っていきます。

### Enableメソッド

`Enable`メソッドはログレベルからログを出力するかを判断するメソッドです。
`Enable`メソッドは単純にラップしたハンドラの`Enable`メソッドを使用するように宣言します。

```go
func (h *JSONIndentHandler) Enabled(ctx context.Context, level slog.Level) bool {
	return h.handler.Enabled(ctx, level)
}
```

### Handleメソッド

`Handle`メソッドはレコードを受け取り、実際にログを出力するメソッドです。
`Handle`メソッドは次のように宣言します。

```go
func (h *JSONIndentHandler) Handle(ctx context.Context, record slog.Record) error {
	h.mu.Lock()
	defer h.mu.Unlock()

	if err := h.handler.Handle(ctx, record); err != nil {
		return err
	}

	encoder := json.NewEncoder(h.w)
	encoder.SetIndent("", strings.Repeat(" ", 2))
	if err := encoder.Encode(json.RawMessage(h.buf.Bytes())); err != nil {
		return fmt.Errorf("failed to encode json log entry: %w", err)
	}
	h.buf.Reset()
	return nil
}
```

`h.handler.Handle`メソッドの呼び出しで`h.buf`にインデントなしのJSONデータが書き込まれます。 
そして、そのデータを`encoder.Encode`メソッドでインデント付きのJSONデータに変換し、`h.w`に書き込みます。
最後に書き込み済みのJSONデータはバッファに必要ないので、`h.buf.Reset`メソッドでバッファから削除します。

重要な点は`Handle`メソッドの開始時に`h.mu.Lock`メソッドでロックを取得している点です。
`w`と`buf`はハンドラ間で共有されており、この2つのフィールドへの書き込みを単純に行えばデータ競合が発生する可能性があります。
そのため、ミューテックスを使用して排他制御を行なっています。

### WithAttrsメソッド、WithGroupメソッド

`WithAttrs`メソッドは指定した属性を含むログを出力する新しいハンドラを返すメソッドで、`WithGroup`メソッドは指定したグループでまとめられたログを出力する新しいハンドラを返すメソッドです。
この2つのメソッドも単純にラップしたハンドラのメソッドを使用して、次のように宣言します。

```go
func (h *JSONIndentHandler) WithAttrs(attrs []slog.Attr) slog.Handler {
	return &JSONIndentHandler{
		handler: h.handler.WithAttrs(attrs),
		w:       h.w,
		mu:      h.mu,
		buf:     h.buf,
	}
}

func (h *JSONIndentHandler) WithGroup(name string) slog.Handler {
	return &JSONIndentHandler{
		handler: h.handler.WithGroup(name),
		w:       h.w,
		mu:      h.mu,
		buf:     h.buf,
	}
}
```

これで`JSONIndentHandler`の実装は完了です。

## 動作確認

実際に`NewJSONIndentHandler`を使ってログを出力してみます。

```go
package main

import (
	"log/slog"
	"os"
)

func main() {
	l := slog.New(NewJSONIndentHandler(os.Stdout, &slog.HandlerOptions{
		AddSource: true,
		ReplaceAttr: func(_ []string, a slog.Attr) slog.Attr {
			if a.Key == slog.MessageKey {
				a.Key = "message"
			}
			return a
		},
	}))

	l.With("a", "b").WithGroup("G").With("c", "d").WithGroup("H").Info("msg", "e", "f")
}
```

上記のプログラムを実行すると次の結果が得られます。
`slog.JSONHandler`と同じフィールドの順序で、インデント付きのJSONデータが出力されることが確認できます。

```json
{
  "time": "2024-07-01T09:00:00.000000+09:00",
  "level": "INFO",
  "source": {
    "function": "main.main",
    "file": "/Users/example/main.go",
    "line": 19
  },
  "message": "msg",
  "a": "b",
  "G": {
    "c": "d",
    "H": {
      "e": "f"
    }
  }
}
```

## テスト

実装した独自のハンドラが適切な出力を行うのかのテストを追加してみます。

独自のハンドラをテストするには標準の`slogtest`パッケージが便利です。
`slogtest`パッケージには独自のハンドラが`slog`のハンドラが満たすべき性質を満たすかを確認するテストケースが用意されています。

例として`JSONIndentHandler`のテストは次のように書けます。

```go
package main

import (
	"bytes"
	"encoding/json"
	"log/slog"
	"testing"
	"testing/slogtest"
)

func TestJSONIndentHandler(t *testing.T) {
	var buf bytes.Buffer

	newHandler := func(t *testing.T) slog.Handler {
		buf.Reset()
		return NewJSONIndentHandler(&buf, nil)
	}
	result := func(t *testing.T) map[string]any {
		line := buf.Bytes()
		if len(line) == 0 {
			return map[string]any{}
		}

		var m map[string]any
		if err := json.Unmarshal(line, &m); err != nil {
			t.Fatal(err)
		}
		return m
	}
	slogtest.Run(t, newHandler, result)
}
```

`slogtest.Run`関数が用意されているテストケースをサブテストで実行する関数で、第2引数、第3引数に`newHandler`関数と`result`関数を受け取ります。
`newHandler`関数はハンドラのインスタンスを生成する関数で、`result`関数はログ出力を`map[string]any`にパースする関数です。

`slogtest.Run`関数はそれぞれのテストケースで次のようにテストを行います。

1. `newHandler`を呼び出してハンドラのインスタンスを生成する
2. 生成したインスタンスでテストケースを実行する
3. テストケースの実行後、`result`関数を呼び出して出力した内容を`map[string]any`で取得し、その内容を検証する

---

これでテストを含めての`JSONIndentHandler`の実装が完了しました。
上記の内容に誤りなどがあればコメントなどで教えていただけると幸いです。

## 参考資料

- [A Guide to Writing slog Handlers](https://github.com/golang/example/blob/master/slog-handler-guide/README.md)
- [slog package](https://pkg.go.dev/log/slog)
- [slogtest package](https://pkg.go.dev/testing/slogtest)
