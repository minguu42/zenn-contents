---
title: "Testcontainersを用いてコンテナを利用するGoアプリケーションのテストを改善する"
emoji: "👼🏼"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["go", "test", "testcontainers"]
published: true
---

## はじめに

GoでMySQLなどのミドルウェアを利用するアプリケーションのテストコードを書く際に、より実際の環境に近い状態でテストするために、モックを使わずにDockerで依存するミドルウェアのコンテナを立てて、テストすることがよくあります。この場合、テストを実行する前にコンテナを起動しておく必要があり、事前準備が手間になってきます。

コンテナに依存するテストパッケージが1つの場合は大丈夫なのですが、複数のテストパッケージがコンテナに依存するようになると、データの衝突を防ぐためにテストを逐次的に実行するか、それとも複数のコンテナを事前に準備するかなどテストを実行するのが大変になってきます。
個人的な思いとしては何も考えずに`go test ./...`を実行してテストを実行できるようにしておきたいです。

このような問題に対して、[Testcontainers for Go](https://golang.testcontainers.org/)が便利だったので、本記事では基本的な使い方を紹介します。Testcontainers for Goは、プログラム上でコンテナの作成・クリーンアップを簡単に行うためのGoパッケージで 、複数のテストパッケージがコンテナに依存していても、簡単にテストパッケージ毎に依存するコンテナを起動でき、事前準備なしでテストを実行できるようになります。

また、似たようなツールとして[dockertest](https://github.com/ory/dockertest)がありますが、[Testcontainersは開発元であるAtomicJar社がDocker社に買収されており](https://www.publickey1.jp/blog/23/dockertestcontainersatomicjardocker.html)、安定した開発が進められると考えられるため、私はTestcontainersを使うことにしました。

この記事が他の人の参考になったら幸いです。
また、この記事の内容に誤った記載がありましたら、指摘してもらえるとありがたいです。

## 使い方

まず、以下のコマンドでTestcontainers for Goをインストールします。

```bash
go get -u github.com/testcontainers/testcontainers-go
```

コンテナを作成するコードは以下のようになります。以下のコード例ではMySQLのコンテナを起動し、起動したコンテナが動作していることを確認しています。実際のテストでは`TestMain`関数でセットした`tdb`変数を使ってデータベースを利用することになると思います。

```go
package main

import (
	"context"
	"database/sql"
	"fmt"
	"log"
	"testing"

	_ "github.com/go-sql-driver/mysql"
	"github.com/testcontainers/testcontainers-go"
	"github.com/testcontainers/testcontainers-go/wait"
)

var tdb *sql.DB

func TestMain(m *testing.M) {
	ctx := context.Background()
	mysqlC, err := testcontainers.GenericContainer(ctx, testcontainers.GenericContainerRequest{
		ContainerRequest: testcontainers.ContainerRequest{
			Image: "mysql:8.0.32",
			Env: map[string]string{
				"MYSQL_DATABASE":             "testdb",
				"MYSQL_ALLOW_EMPTY_PASSWORD": "yes",
			},
			ExposedPorts: []string{"3306/tcp"},
			WaitingFor:   wait.ForListeningPort("3306/tcp"),
		},
		Started: true,
	})
	if err != nil {
		log.Fatalf("failed to create mysql container: %s", err)
	}
	defer func() {
		if err := mysqlC.Terminate(ctx); err != nil {
			log.Fatalf("failed to terminate mysql container: %s", err)
		}
	}()

	host, err := mysqlC.Host(ctx)
	if err != nil {
		log.Fatalf("failed to get host: %s", err)
	}
	port, err := mysqlC.MappedPort(ctx, "3306/tcp")
	if err != nil {
		log.Fatalf("failed to get externally mapped port: %s", err)
	}
	tdb, err = sql.Open("mysql", fmt.Sprintf("%s:%s@tcp(%s:%d)/%s?charset=utf8mb4&parseTime=true",
		"root",
		"",
		host,
		port.Int(),
		"testdb",
	))
	if err := tdb.Ping(); err != nil {
		log.Fatalf("failed to verify a connection to the database: %s", err)
	}

	m.Run()
}
```

上記の`testcontainers.GenericContainer`関数でコンテナを作成しています。`testcontainers.GenericContainerRequest`の`Started`フィールドを`true`にすることでコンテナの作成後、そのまま作成したコンテナを実行します。

コンテナの実行をいつまで待つかは`testcontainers.ContainerRequest`の`WaitingFor`フィールドで制御できます。`wait.ForListeningPort`関数は公開するポートがリッスンできるまで次の処理に進むのを待ちます。他にも`wait.ForLog`関数なども使えますが、`wait.ForLog("ready for connections")`で試していた際に、たまにポートがリッスンされる前に次の処理に進み、テストが落ちることがあったので、`wait.ForListeningPort`関数を使用することをお勧めします。
また、ポートの型は`string`が基底型の`nat.Port`である場合があり、基本的にポートは`80/tcp`の形式で指定する必要があることに気をつけて下さい。

コンテナを実行したら、`Host`メソッドや`MappedPort`メソッドで実行しているコンテナのホストや対応付けされたポートを取得できます。`ExposedPorts`フィールドで公開するポートを指定する場合、ホスト上で対応するポートはdockerdによって選択されるため、テストコードからコンテナを利用するためには`MappedPort`メソッドでホスト上で対応するポートを取得する必要があります。

コンテナのクリーンアップは`Terminate`メソッドによって行われます。上記の例では`defer`によって呼ばれ、パッケージ内の全てのテストケースの終了後にコンテナが削除されます。

他にもコンテナへのマウントを設定したい場合は`testcontainers.ContainerRequest`の`Mounts`フィールドが使用できます。Testcontainers for Goはとても便利なので、皆さんもぜひ使ってください！

## 参考

- [Testcontainers for Go](https://golang.testcontainers.org/)
