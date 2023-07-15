---
title: "Docker で Go + MySQL の環境構築を行った"
emoji: "🦁"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["docker", "go", "mysql"]
published: true
---

## はじめに

この記事では, Docker（Docker Compose）を使って, Go と MySQL の実行環境を作成し, ローカルで Go アプリと MySQL を接続する方法を記述します.

この記事が他の人の参考になれば幸いです.
また, この記事の内容に間違った記載がありましたら, 指摘してもらえるとありがたいです.

## 環境

| 名前          | バージョン |
| ------------- | ---------- |
| macOS Big Sur | 11.4       |
| Docker Engine | 20.10.7    |

## 適当な Go アプリを作成し, 実行する

以下のように単純な Go アプリを作成しました.
この後に `.env`, `Dockerfile`, `docker-compose.yaml` ファイルなどを追加していきます.

```bash:myappのディレクトリ構成
myapp
├── cmd
│   └── server
│       └── main.go
└── go.mod
```

`main.go` は以下のように記述しました.
`PORT` は環境変数として取得するようにしています.

```go:main.go
package main

import (
	"fmt"
	"log"
	"net/http"
	"os"
)

func main() {
	http.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
		_, _ = fmt.Fprint(w, "Hello, 世界！")
	})

	log.Fatal(http.ListenAndServe(":"+os.Getenv("PORT"), nil))
}
```

`go.mod` は以下のように記述しました.

```mod:go.mod
module github.com/minguu42/myapp

go 1.16
```

### `Dockerfile` を作成する

Go の実行環境の Docker イメージを定義する `Dockerfile` を作成します.
公式の Go の Docker Hub に記述されている内容を参考に以下のように記述しました.

`go get -d -v ./...` コマンドで `go.mod` を更新し, Go アプリの依存モジュールをダウンロードしています. `-d` オプションで依存モジュールをインストールしないように指定し, `-v` オプションで詳細なデバッグを表示するように指定しています.

そして, Go アプリは `go run ./cmd/server` コマンドで実行しています.

```dockerfile
FROM golang:1.16

WORKDIR /go/src/app
COPY . .

RUN go get -d -v ./...

CMD ["go", "run", "./cmd/server"]
```

### `.env`, `docker-compose.yaml` を作成する

`.env`, `docker-compose.yaml` ファイルを作成し, Go アプリを実行できるようにします.
環境によって異なる, Git などで管理したくない情報は `.env` ファイルで環境変数として管理します. そのため, Git などを使用している場合は `.env` ファイルは `.gitignore` などで無視するように設定してください.

`.env` ファイルは以下のように記述しました.

```text:.env
PORT=8080
```

`docker-compose.yaml` ファイルは以下のように記述しました.

`.env` ファイルから環境変数 `PORT` を読み込んでいます. また, ポートフォワードを使用し, ローカルからも Docker コンテナにアクセスできるようにしています.

```yaml:docker-compose.yaml
version: "3"

services:
  myapp:
    build:
      context: .
      dockerfile: Dockerfile
    environment:
      - PORT
    container_name: myapp
    stdin_open: true
    tty: true
    volumes:
      - .:/go/src/app
    ports:
      - ${PORT}:${PORT}
```

### Docker Compose で Go アプリを実行する

以下のコマンドで Docker Compose で Go アプリを実行します.

```bash:terminal
docker compose up -d
```

以下のコマンドで Go アプリが立ち上がっているのを確認します.

```bash:terminal
$ curl http://localhost:8080
Hello, 世界！%
```

## MySQL の実行環境を作成し, Go アプリと接続する

### `.env`, `docker-compose.yaml` ファイルを変更する

`.env` ファイルを以下のように変更し, 環境変数を設定します.
環境変数 `DRIVER`, `DSN` は Go アプリ側で使用する DB の接続情報です.
環境変数 `MYSQL_ROOT_PASSWORD` などは MySQL イメージの初期化で使用する設定情報です.
`DSN` の各値は `MYSQL_DATABASE`, `MYSQL_USER`, `MYSQL_PASSWORD` に対応するように気をつけてください.

```text:.env
PORT=8080
DRIVER=mysql
DSN=user:password@tcp(db-dev:3306)/db_dev

MYSQL_ROOT_PASSWORD=root_password
MYSQL_DATABASE=db_dev
MYSQL_USER=user
MYSQL_PASSWORD=password
```

`docker-compose.yaml` ファイルを以下のように変更し, MySQL の実行環境を設定します.
MySQL のサービスでは名前付きボリュームを使用し, MySQL のデータをローカルのボリュームに保存しています. また, MySQL で初期化時に SQL ファイルなどを実行したい場合は, `./example.sql:/docker-entrypoint-initdb.d/example.sql` のように `volumes:` に追記し, Docker コンテナ側の `/docker-entrypoint-initdb.d` ディレクトリにマウントすると実行されます.

```yaml:docker-compose.yaml
version: "3"

services:
  myapp:
    build:
      context: .
      dockerfile: Dockerfile
    environment:
      - PORT
      - DRIVER
      - DSN
    container_name: myapp
    stdin_open: true
    tty: true
    volumes:
      - .:/go/src/app
    ports:
      - ${PORT}:${PORT}
    depends_on:
      - db-dev
  db-dev:
    image: mysql:8.0
    environment:
      - MYSQL_ROOT_PASSWORD
      - MYSQL_DATABASE
      - MYSQL_USER
      - MYSQL_PASSWORD
    container_name: db-dev
    restart: always
    volumes:
      - data-dev:/var/lib/mysql

volumes:
  data-dev:
    driver: local
```

### Go アプリのソースコードの変更

MySQL に接続できるように Go アプリのコードを変更します.
プロジェクトディレクトリにデータベースに関する処理を行う `db.go` を以下のように作成しました.

```go:db.gp
package myapp

import (
	"database/sql"
	"log"
)

func OpenDB(driver, dsn string) *sql.DB {
	db, err := sql.Open(driver, dsn)
	if err != nil {
		log.Fatal("OpenDB failed:", err)
	}
	return db
}

func CloseDB(db *sql.DB) {
	if err := db.Close(); err != nil {
		log.Fatal("CloseDB failed:", err)
	}
}
```

`main.go` を以下のように変更しました.
注意点として `sql.Open` メソッドではなく, `db.Ping` メソッドで DB との接続を確認できます.
また, MySQL のコンテナが立ち上がり, サーバが起動するまでに `db.Ping` メソッドが実行されるとエラーが出るので, `db.Ping` メソッドはエラーが発生した場合は時間を空けて再度実行しています.

```go:main.go
package main

import (
	"fmt"
	"log"
	"net/http"
	"os"

	_ "github.com/go-sql-driver/mysql"
	"github.com/minguu42/myapp"
)

func main() {
	db := myapp.OpenDB(os.Getenv("DRIVER"), os.Getenv("DSN"))
	defer myapp.CloseDB(db)

	if err := db.Ping(); err != nil {
		log.Fatal("db.Ping failed:", err)
	}

	http.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
		_, _ = fmt.Fprint(w, "Hello, 世界！")
	})

	log.Fatal(http.ListenAndServe(":"+os.Getenv("PORT"), nil))
}
```

### Go アプリと MySQL を実行する

以下の一連のコマンドを実行し, Go アプリと MySQL を実行します.
`go mod tidy` はソースコードから依存モジュールを検出し, `go.mod` に反映するために実行します.

```bash:terminal
go mod tidy
docker compose up -d
```

以下のコマンドで Go アプリが立ち上がっていることを確認できます.

```bash:terminal
$ curl http://localhost:8080
Hello, 世界！%
```

## 参考

- [Golang - Official Image | DockerHub](https://hub.docker.com/_/golang)
- [MySQL - Official Image | DockerHub](https://hub.docker.com/_/mysql)
