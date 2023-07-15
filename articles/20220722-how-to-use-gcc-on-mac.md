---
title: "MacでGCCを使用し、C/C++のコードをコンパイルする"
emoji: "😎"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["c", "cpp"]
published: true
---

## はじめに

こんにちは！[minguu42](https://twitter.com/minguu42)です。
MacではCのコンパイラとして標準でClangが入っているのですが、ClangではなくGCCを使いたい場合もあると思います。
この記事では、GCCでコンパイルするための設定をメモとして残します。

この記事が他の人の参考になったら幸いです。
また、この記事の内容に誤った記載がありましたら、指摘してもらえるとありがたいです。

## GCCのインストール

最初にHomebrewでGCCをインストールします。

```bash:terminal
brew install gcc
```

## パスの設定

HomebrewでGCCをインストールしてもパスの設定を行わないと`gcc`コマンドや`g++`コマンドでClangが呼び出されます。

```bash:terminal
$  gcc --version
Apple clang version 13.1.6 (clang-1316.0.21.2.5)
Target: x86_64-apple-darwin21.5.0
Thread model: posix
InstalledDir: /Library/Developer/CommandLineTools/usr/bin

$  g++ --version
Apple clang version 13.1.6 (clang-1316.0.21.2.5)
Target: x86_64-apple-darwin21.5.0
Thread model: posix
InstalledDir: /Library/Developer/CommandLineTools/usr/bin
```

そのため、`/usr/local/bin/`にGCCのコマンドへのシンボリックリンクを作成し、`gcc`コマンドや`g++`コマンドで呼び出せるように変更します。

まず、HomebrewでインストールしたGCCのバイナリはそれぞれ以下のように確認できます。

```bash:terminal
$  ls /usr/local/bin | grep gcc
gcc-11
gcc-ar-11
gcc-nm-11
gcc-ranlib-11
x86_64-apple-darwin21-gcc-11
x86_64-apple-darwin21-gcc-ar-11
x86_64-apple-darwin21-gcc-nm-11
x86_64-apple-darwin21-gcc-ranlib-11

$  ls /usr/local/bin | grep g++
g++-11
x86_64-apple-darwin21-g++-11
```

そして、`/usr/local/bin/`に使用したいバイナリに対するシンボリックリンクを作成します。

```bash:terminal
$ ln -s /usr/local/bin/gcc-11 /usr/local/bin/gcc
$ ln -s /usr/local/bin/g++-11 /usr/local/bin/g++
```

一旦シェルを落として立ち上げ直すと、`gcc`コマンドと`g++`コマンドがGCCを参照するようになります！これで本記事でやりたかったことは完了です。

```bash:terminal
$ exec $SHELL -l # シェルの再起動

$ gcc --version
gcc (Homebrew GCC 11.3.0_2) 11.3.0
Copyright (C) 2021 Free Software Foundation, Inc.
This is free software; see the source for copying conditions.  There is NO
warranty; not even for MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.

$ g++ --version
gcc (Homebrew GCC 11.3.0_2) 11.3.0
Copyright (C) 2021 Free Software Foundation, Inc.
This is free software; see the source for copying conditions.  There is NO
warranty; not even for MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.
```

また、`gcc`コマンドや`g++`コマンドで呼び出されるコンパイラをClangに戻したい場合はシンボリックリンクを削除すれば大丈夫です。

```bash:terminal
$ unlink /usr/local/bin/gcc
$ unlink /usr/local/bin/g++
```

## 参考

- [Visual studio code で競プロ環境構築[mac OS]](https://qiita.com/EngTks/items/ffa2a7b4d264e7a052c6)
