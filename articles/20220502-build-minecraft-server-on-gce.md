---
title: "Google Compute EngineでMinecraftサーバを構築する"
emoji: "😊"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["gcp", "minecraft"]
published: true
---

## はじめに

こんにちは！[minguu42](https://twitter.com/minguu42)です。
最近、友達とマルチプレイヤーでMinecraftがやりたいという話になり、インフラに関する知識はほとんどないですが、マインクラフトのサーバを構築することになりました。
この記事では、Google Compute Engineを使って、Minecraftサーバを構築する方法や学んだことを書きます。

この記事が他の人の参考になったら幸いです。
また、この記事の内容に誤った記載がありましたら、指摘してもらえるとありがたいです。

## 環境

| 名前             | バージョン |
| ---------------- | ---------- |
| macOS Monterey   | 12.3.1     |
| Google Cloud SDK | 383.0.1    |

## Compute Engineの仮想マシンのリソースを決める

### Minecraftサーバに必要なリソースはどのくらいか？

まず、Minecraftサーバにどのくらいのリソースが必要か調べました。
[さくらのVPSのおすすめ](https://manual.sakura.ad.jp/vps/startupscript/minecraft-java.html)では以下のようになっていました。

| 人数   | CPU | メモリ（GB） | SSD（GB）    |
| ------ | --- | ------------ | ------------ |
| 1 ~ 4  | 2   | 1            | 50 ~ 100     |
| 5 ~ 10 | 4   | 4            | 200 ~ 400 GB |
| 10 ~   | 6   | 8            | 400 ~ 800    |

[ConoHa VPSの推奨プラン](https://www.conoha.jp/vps/function/minecraft/)では以下のようになっていました。

| 人数   | CPU | メモリ（GB） | SSD（GB） |
| ------ | --- | ------------ | --------- |
| 1 ~ 4  | 3   | 2            | 100       |
| 5 ~ 10 | 4   | 4            | 100       |
| 10 ~   | 6   | 8            | 100       |

この情報を参考にしながら、Compute Engineの料金と見比べて、構築するサーバのリソースを決定します。

### Compute Engineのリソースと料金

この記事では Compute Engine上にMinecraftサーバを構築します。
Compute Engineは仮想マシンを作成して実行できるコンピューティング、およびホスティングサービスです。
今回、Compute Engine上にMinecraftサーバを構築するのはCompute Engineが従量課金制で利用できるからです。ゲームをするときだけ起動してそれ以外のときは停止します。

今回の用途からマシンファミリーは`汎用`、マシンシリーズは`E2`、`N1`を考えました。
`E2`、`N1`の用意されているマシンタイプのリソースと料金は以下のようになります。
注意点として以下の表の料金はGCP Consoleで月間予想として表示される料金であり、リージョンなどによって変わる異なる可能性があるので参考程度に見てください。

| マシンタイプ  | vCPU | メモリ（GB） | 料金            | 永続ディスクの最大数 | 永続ディスクの最大合計サイズ（TB） | 最大下り帯域幅（Gbps） |
| ------------- | ---- | ------------ | --------------- | -------------------- | ---------------------------------- | ---------------------- |
| e2-micro      | 0.25 | 1            | $9.14 ($0.01)   | 16                   | 3                                  | 1                      |
| e2-small      | 0.5  | 2            | $16.99 ($0.02)  | 16                   | 3                                  | 1                      |
| e2-medium     | 1    | 4            | $31.38 ($0.04)  | 16                   | 3                                  | 2                      |
| e2-standard-2 | 2    | 8            | $64.05 ($0.09)  | 128                  | 257                                | 4                      |
| e2-standard-4 | 4    | 16           | $126.81 ($0.17) | 128                  | 257                                | 8                      |
| e2-highcpu-2  | 2    | 2            | $47.68 ($0.07)  | 128                  | 257                                | 4                      |
| e2-highcpu-4  | 4    | 4            | $94.06 ($0.13)  | 128                  | 257                                | 8                      |
| e2-highcpu-8  | 8    | 8            | $186.81 ($0.26) | 128                  | 257                                | 4                      |
| n1-standard-1 | 1    | 3.75         | 32.44 ($0.04)   | 128                  | 257                                | 2                      |
| n1-standard-2 | 2    | 7.50         | 63.58 ($0.09)   | 128                  | 257                                | 10                     |
| n1-standard-4 | 4    | 15           | 125.86 ($0.17)  | 128                  | 257                                | 10                     |

私はMinecraftサーバに必要なリソースと、Compute Engineのリソースと料金の兼ね合いから`e2-highcpu-2`を使用することにしました。

#### 参考

- [Compute Engine ドキュメント](https://cloud.google.com/compute/docs)
- [マシン ファミリーについて](https://cloud.google.com/compute/docs/machine-types)
- [汎用マシン ファミリー](https://cloud.google.com/compute/docs/general-purpose-machines)

## Minecraftサーバを構成する

ここからはGCPのCLIツール`gcloud`を使って、サーバを実際に構築します。
GCPのプロジェクトは作成済みで必要なAPIも有効済みであり、`gcloud`のセットアップも終わっているものとして進めます。

### Compute Engineの仮想マシンを作成する

まず、以下のコマンドで外部静的IPアドレス（グローバルIPアドレス）を作成します。
Minecraftサーバに外部からアクセスするために必要です。

```bash
gcloud compute addresses create minecraft-ip
```

以下のコマンドで、作成したアドレスのIPアドレスを確認しておいて下さい。

```bash
gcloud compute addresses describe minecraft-ip
```

そして、以下のコマンドでCompute Engineの仮想マシンを作成します。
`--adress`には上記で確認したIPアドレスを使用します。
`--tag`はファイアウォールを設定するのに使用します。

```bash
gcloud compute instances create minecraft-server \
  --image=ubuntu-2204-jammy-v20220420 \
  --image-project=ubuntu-os-cloud \
  --machine-type=e2-highcpu-2 \
  --address=<IP アドレス> \
  --tags=minecraft-net
```

作成した仮想マシンに対して、ファイアウォールルールを作成します。
このファイアウォールルールは任意のIPv4アドレス（`0.0.0.0/0`）からの`25565`ポートへの`TCP`もしくは`UDP`プロトコルの通信のみを許可します。
Minecraftサーバとクライアントはデフォルトの場合に`25565`ポートで通信を行うので、その通信のみを許可するようにしています。

```bash
gcloud compute firewall-rules create minecraft-firewall \
  --priority=1000 \
  --direction=ingress \
  --action=allow \
  --rules=tcp:25565,udp:25565 \
  --source-ranges=0.0.0.0/0 \
  --target-tags=minecraft-net
```

#### 参考

- [VM インスタンスの作成と起動](https://cloud.google.com/compute/docs/instances/create-start-instance)
- [静的外部 IP アドレスの予約](https://cloud.google.com/compute/docs/ip-addresses/reserve-static-external-ip-address)
- [ファイアウォール ルールの使用](https://cloud.google.com/vpc/docs/using-firewalls)

### 仮想マシンに接続し、Minecraftサーバを構築する

以下のコマンドで作成した仮想マシンに対してSSH接続できます。
事前にSSHキーを作成していなくても対話的に自動で生成できます。

```bash
gcloud compute ssh "minecraft-server"
```

仮想マシンに接続できたら、Minecraftのサーバを構築していきます。
`server.jar`は[公式](https://www.minecraft.net/ja-jp/download/server)よりダウンロードします。
サーバのファイル名はバージョンが分かるように名前を`server.1.18.2.jar`に変更しています。
`eula.txt`はEULA(https://account.mojang.com/documents/minecraft_eula)に同意したことを示すためのファイルでサーバを起動するのに必要です。

`screen -d -m`コマンドによってデタッチされたウィンドウでサーバが起動されます。
もし、そのウィンドウにアタッチしたい場合は`screen -r minecraft`でアタッチできます。
現在のウィンドウをデタッチしたい場合はキーマップ`ctrl-a d`で出来ます。
また、`screen -ls`で起動しているウィンドウを見れます。

```bash
$ sudo apt update && sudo apt upgrade

# Java のインストール
$ sudo apt install openjdk-18-jre-headless

# Minecraft: Java Edition のサーバをダウンロード
$ mkdir ~/minecraft && cd ~/minecraft
$ curl -Ol https://launcher.mojang.com/v1/objects/c8f83c5655308435b3dcf03c06d9fe8740a77469/server.jar
$ mv server.jar server.1.18.2.jar
$ echo "eula=true" >> eula.txt

# Minecraft サーバを起動する
$ screen -S minecraft -d -m java -Xmx1024M -Xms1024M -jar server.1.18.2.jar nogui
```

起動したサーバを停止したい場合はサーバを起動しているウィンドウにアタッチして`stop`とコマンドを打ちます。

#### 参考

- [Linux VMへの接続](https://cloud.google.com/compute/docs/instances/connecting-to-instance)

### Minecraftサーバに接続する

サーバが起動できたので、Minecraftからサーバに接続します。
Minecraft Java Editionを開き、`Multiplayer > Add Server`からサーバを追加します。
`Server Name`に任意の名前を入れて、`Server Address`にCompute Engineの作成時に`--adress`に指定した値を入れます。
`Server Resource Packs`は`Prompt`のままで`Done`をクリックすると接続できます。
