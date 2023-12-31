---
title: "HerokuにGoアプリをデプロイした"
emoji: "🎉"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["go", "heroku"]
published: true
---

## はじめに

この記事ではHerokuにGoアプリをデプロイする方法について書きます。

この記事では以下を前提としています。

- Heroku CLIをインストールし、ログインしている
- GoアプリはGitで管理している
- Goアプリの依存管理はGo Modulesを使用している

この記事が他の人の参考になれば幸いです。
また、この記事の内容に間違った記載がありましたら、指摘してもらえるとありがたいです。

## 環境

| 名前          | バージョン |
| ------------- | ---------- |
| macOS Big Sur | 11.5       |
| Go            | 1.16.5     |
| Heroku CLI    | 7.56.1     |

## デプロイ

### 最初のデプロイ

初回のHerokuへのデプロイは以下の手順で行います。

1. `go.mod`でHerokuで使用するGoのバージョンを指定する
2. `Procfile`を作成する
3. Heroku CLIでデプロイする

まず、`go.mod`ファイルにHerokuで使用するGoのバージョンを指定します。
`// +heroku goVersion go1.16`のコメントで指定しています。

```go:go.mod
module github.com/minguu42/myapp

go 1.16
// +heroku goVersion go1.16

...
```

次に、`Procfile`を作成します。
`Procfile`では実行するバイナリを指定します。
バイナリが一つの場合などでは`Procfile`を作成しなくてもHerokuが自動でバイナリを選択しますが、ローカルでの実行時などには必要なので作成します。
myappは自分のmainパッケージ名に合わせて変更してください。

```text:Procfile
web: bin/myapp
```

最後に以下の一連のコマンドでデプロイできます。
Goアプリのビルドや実行などはHeroku側で自動的に行ってくれます。

```bash:terminal
# Heroku アプリを作成する
heroku create [appName]

# Heroku にデプロイする
git push heroku main
```

### 以後のデプロイ

初回のデプロイ後は以下のコマンドでデプロイできます。

```bash:terminal
git push heroku main
```

## 環境変数

Heroku環境での環境変数は以下のコマンドで扱えます。

### 環境変数を一覧表示

```bash:terminal
heroku config
```

### 環境変数の追加

`KEY`と`VALUE`は適切な値に変更してください。

```bash:terminal
heroku config:set KEY=VALUE
```

### 環境変数の削除

```bash:terminal
heroku config:unset KEY
```

## データベース

Herokuではプラグインを使用し、データベースを扱えます。
ここではMySQLを扱う方法を見ていきます。

以下のコマンドでMySQLのプラグインを追加します。

```bash
heroku addons:create jawsdb:kitefin
```

コマンドの実行後は複数の環境変数が追加されます。
GoアプリのDBと接続するコードで環境変数を読み込み、DBと接続するようにして下さい。

実際にデータベースが立ち上がっているかは以下のコマンドで接続し、確認できます。
`user`、`password`、`host`、`database`は上記のコマンドで追加された環境変数から確認できます。

```bash:terminal
mysql -u <user> -p -h <host> -D <database>
```

## main ブランチ以外のブランチでデプロイしたい

実行に必要な秘密ファイルをGitHubに上げたくないがHerokuには上げたい場合などmainブランチ以外のブランチをデプロイしたい場合はあると思います。
Herokuはmainブランチのみをビルドし、デプロイするのでそのままプッシュしてもデプロイできません。
以下のコマンドでHerokuにプッシュしてデプロイできます。

```bash:terminal
git push heroku <branch name>:main
```

## Heroku CLIの便利なコマンド

### ローカルで実行する

ローカルでビルドし、`Procfile`に従って実行します。
環境変数は`.env`ファイルを読み込みます。
myappは適宜変更して下さい。

```bash:terminal
# Go アプリをビルドする
go build -o bin/myapp -v .

# Heroku をローカルで実行する
heroku local web
```

### ログを表示する

```bash:terminal
heroku logs --tail
```

### Herokuアプリの名前を変更する

```bash:terminal
heroku apps:rename <appName>
```

## 参考

- [Getting Started on Heroku with Go](https://devcenter.heroku.com/articles/getting-started-with-go?singlepage=true)
- [JawsDB MySQL - Add-ons - Heroku Elements](https://elements.heroku.com/addons/jawsdb)
