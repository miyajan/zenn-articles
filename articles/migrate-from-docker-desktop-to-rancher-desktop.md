---
title: "Mac で Docker Desktop から Rancher Desktop へ移行する"
emoji: "🐳"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["docker", "rancher", "mac"]
published: true
---

## 概要

この記事では、Mac で Docker Desktop から Rancher Desktop への移行について調査した内容をまとめます。

## 背景知識

### なぜ Mac で Docker Desktop (or Rancher Desktop) が必要なのか？

語弊を覚悟で書くと、現状では Docker は基本的には Linux 上で動かすツールです。なので、Linux 以外の OS で Docker を利用するためにはなにかしらの方法で Linux を OS 上で動作させる必要があります。

Windows の場合は WSL という Linux カーネルを動作させる仕組みがありますが、Mac にはありません。

このため、Mac では Linux VM を立ち上げ、ホスト側で VM 上の `/var/run/docker.sock` をマウントし、このソケット経由で Docker デーモンに命令するという方法が主流となっています。

これを手軽に実現できるようにしたのが、Docker Desktop（や Rancher Desktop）というアプリケーションになります。

### Rancher Desktop とは

https://rancherdesktop.io/

以下は、[Rancher Desktop の Introduction にある Rancher vs Rancher Desktop](https://docs.rancherdesktop.io/#rancher-vs-rancher-desktop) からの引用です。

> While Rancher and Rancher Desktop share the Rancher name they do different things. Rancher Desktop is not Rancher on the Desktop. Rancher is a powerful solution to manage Kubernetes clusters. Rancher Desktop provides a local Kubernetes and container management platform. The two solutions complement each other.
> If you want to run Rancher on your local system, you can install Rancher into Rancher Desktop.

要約すると、Rancher Desktop は Rancher のデスクトップ版**ではありません**。Rancher は Kubernetes クラスタを管理するためのプラットフォームですが、Rancher Desktop はローカルの Kubernetes やコンテナを管理するためのアプリケーションです。

Rancher Desktop の内部的な仕組みは、以下のようになります。（[Rancher Desktop ホームページ](https://rancherdesktop.io/)の "How it works" より引用）

![Rancher Desktop のアーキテクチャ図](/images/migrate-from-docker-desktop-to-rancher-desktop-on-mac/how-it-works-rancher-desktop.png)

Mac の場合、Rancher Desktop は内部的には [Lima](https://github.com/lima-vm/lima) を使って VM を立ち上げます。

## Docker Desktop から Rancher Desktop への移行

筆者は、以下の環境で調査しました。

- Mac
  - OS: Monterey 12.6
  - CPU: M1 Pro
- Rancher Desktop: 1.6.1

### Docker Desktop のアンインストール

まず、[公式ドキュメント](https://docs.docker.com/desktop/uninstall/)に従って、Docker Desktop の `Troubleshoot` から `Uninstall` します。（ゴミデータが残らないように事前に `Clean / Purge data` しておいたほうがいいという記事も見かけるので、気になる人はアンインストール前に実行しておくといいかもしれません）

アンインストールを実行しても、Mac の `Applications` ディレクトリに Docker のアプリケーションが残ったままになるので、手動でゴミ箱に入れておきましょう。

### Rancher Desktop のインストール

基本的には[公式ドキュメント](https://docs.rancherdesktop.io/getting-started/installation/)に従います。

まず、[Rancher Desktop のリリースページ](https://github.com/rancher-sandbox/rancher-desktop/releases)から、アーカイブをダウンロードします。自分は M1 Mac なので、"Rancher.Desktop-1.6.1.aarch64.dmg" をダウンロードしました。

アーカイブを実行すると Mac おなじみのアイコンを "Applications" ディレクトリに D&D するダイアログが表示されるので、D&D します。

### Rancher Desktop の初期設定

Rancher Desktop アプリケーションを開くと、以下のように `Welcome to Rancher Desktop` という初期設定ダイアログが表示されます。

![Rancher Desktop の初期設定ダイアログ](/images/migrate-from-docker-desktop-to-rancher-desktop-on-mac/welcome-to-rancher-desktop.png)

`Enable Kubernetes` は、ローカルで Kubernetes クラスタを動かす必要がないならチェックを外しましょう。

`containerd` と `dockerd (moby)` は、これまで通り Docker CLI を使いたいのであれば、`dockerd (moby)` を選択しましょう。（`containerd` を選択すると、Docker CLI の代わりに `nerdctl` コマンドを使うことになります）

`Configure PATH` は、`Automatic` にすると利用中のシェルのプロファイル（e.g. `$HOME/.zshrc`）に、`PATH` に `$HOME/.rd/bin` を追加する設定が自動で入ります。基本的には `Automatic` で困りませんが、`PATH` の順番を意図的に変更したい場合などは `Manual` を選択し、自分でシェルのプロファイルを設定して `PATH` に `$HOME/.rd/bin` を追加しましょう。（自分の場合、`asdf` でインストールする `kubectl` が優先的に選ばれてほしかったので、自分でシェルのプロファイルを修正しました）

![管理権限を要求されるダイアログ](/images/migrate-from-docker-desktop-to-rancher-desktop-on-mac/administrative-access-required.png)

初期設定が終わると、`Administrative Access Required` という管理権限を要求するダイアログが表示されます。`Always run without administrative access` のチェックは外したままで `OK` を押し、Mac のパスワードを入力しましょう。

### 動作確認

初期設定が終わると、Rancher Desktop が立ち上がり、`Starting virtual machine` という VM を起動するメッセージなどが表示されます。起動時のメッセージが消えたら、ターミナルを立ち上げ、`docker version` コマンドでも叩いて動作確認してみます。

```bash
$ docker version
Client:
 Version:           20.10.17-rd
 API version:       1.41
 Go version:        go1.17.11
 Git commit:        c2e4e01
 Built:             Fri Jul 22 18:32:57 2022
 OS/Arch:           darwin/arm64
 Context:           default
 Experimental:      true

Server:
 Engine:
  Version:          20.10.18
  API version:      1.41 (minimum version 1.12)
  Go version:       go1.18.6
  Git commit:       e42327a6d3c55ceda3bd5475be7aae6036d02db3
  Built:            Sun Sep 11 07:10:00 2022
  OS/Arch:          linux/arm64
  Experimental:     false
 containerd:
  Version:          v1.6.8
  GitCommit:        9cd3357b7fd7218e4aec3eae239db1f68a5a6ec6
 runc:
  Version:          1.1.4
  GitCommit:        5fd4c4d144137e991c4acebb2146ab1483a97925
 docker-init:
  Version:          0.19.0
  GitCommit:        
```

無事動きました。

あとは、普段の自分の開発で使うコマンドなどを試して、問題なく動作することを確認してみましょう。

## 既知の問題

### 大量のログファイルが生成されてディスクフルになる

自分は遭遇していないですが、同じチームのメンバーによると、大量のログファイルが生成されてディスクフルになる問題があるとのことです。再現条件は不明ですが、もしこの現象が起きたら factory reset すると解消するようです。

https://twitter.com/Shitimi_613/status/1585565665519599616

[関連 issue](https://github.com/rancher-sandbox/rancher-desktop/issues/1942)

## まとめ

Mac で Docker Desktop から Rancher Desktop への移行について調査した内容をまとめました。筆者が試してる限りでは今のところ特に問題に遭遇しておらず、思ったよりスムーズに移行できています。

この記事は、必ずしも Docker Desktop から Rancher Desktop への移行を推奨するものではありません。現状では Rancher Desktop は枯れたツールではないため、ユースケースによっては想定外の問題にハマることもあると思われます。自分も今後問題に遭遇することがあれば、記事に追記する予定です。
