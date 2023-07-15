---
title: "GitHub Actions に入門した"
emoji: "💬"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["githubactions"]
published: true
---

## はじめに

この記事は GitHub Actions を触ってみて理解したことを備忘録代わりにまとめたものです.
主に GitHub Actions とは何か, 始めるための最低限のセットアップ方法, 自分が GitHub Actions の基礎として理解しておきたいことを書きます.
そのため, GitHub Actions に興味があるが触ったことがない人, 触り始めた人向けになります.

この記事が他の人の参考になれば幸いです.
また, この記事の内容に間違った記載がありましたら, 指摘してもらえるとありがたいです.

## GitHub Actions とは何か?

`GitHub Actions` は GitHub が提供する CI/CD サービスです. GitHub のリポジトリに対して何らかのイベント（プッシュなど）が起こった時に一連のコマンドを実行できます.

### GitHub Actions の基本概念

GitHub Actions で使われる概念や言葉を説明しておきます.
`ワークフロー(Workflows)` は 1 つ以上のジョブを含み, イベントによって実行される自動化された手続きです.
`イベント(Events)` はワークフローを実行するトリガーとなる特定の活動です. 外部のイベントもトリガーにできます.
`ジョブ(Jobs)` は同じランナーで実行される一連のステップです. 1 つのワークフローに複数のジョブがある場合は, デフォルトではジョブはそれぞれ**並列**で実行されます.
`ステップ(Steps)` はジョブに含まれる個別のアクション, もしくは, シェルコマンドです. ジョブのそれぞれのステップは同じランナーで実行されるのでステップ間で同じデータが使用できます.
`アクション(Actions)` は他のワークフローでも使用できる独立したコマンドです.
`ランナー(Runners)` はジョブを実行するサーバです. GitHub がホストする, もしくは, 自分でホストするランナーが使用できます. ランナーは新しい仮想環境でジョブを実行し, ログや結果を記録して GitHub に返します. これはワークフロー内に複数のジョブがある場合でも同様でそれぞれのジョブで新しい仮想環境が作成され, 実行されます.

GitHub Actions が一連のコマンドを実行する流れは大まかに以下のようになります.

1. イベントによりワークフローが実行される.
2. ワークフローはジョブを並列に実行する.
3. ランナーでジョブに含まれるステップが順番に実行される.
4. ランナーが結果を GitHub に返し, 確認できる

詳しくは [Understanding GitHub Actions](https://docs.github.com/en/actions/learn-github-actions/understanding-github-actions) を参照してください.

## 始め方

GitHub Actions は以下の手順で始められます.
前提として GitHub Actions を始める前には GitHub リポジトリが必要になります.

1. `.github/workflows` ディレクトリを作成する
2. そのディレクトリ内に任意のワークフローファイルを作成する

実際にやってみたい場合は公式チュートリアル [Quickstart for GitHub Actions](https://docs.github.com/en/actions/quickstart) が手軽でおすすめです.

## ワークフローファイルの作成

ワークフローを作成するために `.github/workflows` ディレクトリに任意のワークフローファイルを作成します. ワークフローファイルは YAML で記述します.

```yaml:github-actions-example.yaml
# ワークフロー名の設定
name: github-actions-example

# トリガーの設定
# この場合は backend ディレクトリ配下に変更が加えたコミットが main ブランチにプッシュされた場合にワークフローを実行する.
on:
  push:
    branches:  # パターンとブランチ名が一致した場合に実行する
      - main
    paths:     # パターンとパスが一致した場合に実行する
      - backend/**

# ジョブの設定
jobs:
  example-job:                                 # 一意のジョブ ID
    runs-on: ubuntu-20.04                      # ランナーの設定
    env:                                       # ステップ間で共有する環境変数の設定
      GREETING: Hello
    steps:
      - name: Checkout code                    # ステップ名の設定
        uses: actions/checkout@v2              # アクションを実行する
      - name: Set up Go
        uses: actions/setup-go@v2
        with:
          go-version: 1.17.1
      - name: Show Go version
        run: go version                        # コマンドを実行する
        working-directory: ./backend           # ワーキングディレクトリの設定
      - name: Greeting
        run: echo "$GREETING $FIRST_NAME $LAST_NAME"
        env:                                   # 環境変数の設定
          FIRST_NAME: Satoshi
          LAST_NAME: ${{ secrets.LAST_NAME }}  # 秘密情報の読み出し
  other-job:
    ...
```

大まかな書き方は上の例のようになります.
`${{ secrets.LAST_NAME }}` は事前に GitHub リポジトリの `Settings > Secrets` ページで設定した秘密情報 `LAST_NAME` を読み出しています.

より詳しい情報は以下の公式ドキュメントに書かれています.

- [Essential features of GitHub Actions](https://docs.github.com/en/actions/learn-github-actions/essential-features-of-github-actions) ... 環境変数の使い方, Bash スクリプトの実行方法などについて
- [Managing complex workflows](https://docs.github.com/en/actions/learn-github-actions/managing-complex-workflows) ... 秘密情報の扱い方, 直列で実行する方法などについて
- [Events that trigger workflows](https://docs.github.com/en/actions/learn-github-actions/events-that-trigger-workflows) ... 様々なイベントについて
- [Expressions](https://docs.github.com/en/actions/learn-github-actions/expressions) ... 動的な値の設定について
- [Workflow syntax for GitHub Actions](https://docs.github.com/en/actions/learn-github-actions/workflow-syntax-for-github-actions) ... 詳細な文法, 設定項目について
- [Environment variables](https://docs.github.com/en/actions/learn-github-actions/environment-variables) ... 環境変数について

## 参考

- [Quickstart for GitHub Actions](https://docs.github.com/en/actions/quickstart)
- [Understanding GitHub Actions](https://docs.github.com/en/actions/learn-github-actions/understanding-github-actions)
- [Essential features of GitHub Actions](https://docs.github.com/en/actions/learn-github-actions/essential-features-of-github-actions)
- [Managing complex workflows](https://docs.github.com/en/actions/learn-github-actions/managing-complex-workflows)
- [Events that trigger workflows](https://docs.github.com/en/actions/learn-github-actions/events-that-trigger-workflows)
- [Expressions](https://docs.github.com/en/actions/learn-github-actions/expressions)
- [Workflow syntax for GitHub Actions](https://docs.github.com/en/actions/learn-github-actions/workflow-syntax-for-github-actions)
- [Environment variables](https://docs.github.com/en/actions/learn-github-actions/environment-variables)
