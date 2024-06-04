---
title: "GitHub Actions に Arm64 ランナーが来たので Docker のマルチプラットフォームイメージをビルドしてみる"
emoji: "🐳"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["githubactions", "docker"]
published: true
---

2024/06/03 に GitHub Actions に Arm64 ランナーが追加されました。

@[card](https://github.blog/2024-06-03-arm64-on-github-actions-powering-faster-more-efficient-build-systems/)

現在はパブリックベータで、Team と Enterprise Cloud プランでのみ利用可能です。料金は x64 の同性能のランナーより 37% 安く、電力効率が高いため二酸化炭素排出量削減にもつながるとのことです。

この記事では、新しく追加された Arm64 ランナーを使って Docker のマルチプラットフォームイメージをビルドしてみます。

## マルチプラットフォームイメージとは？

マルチプラットフォームイメージとは、**複数の異なる CPU アーキテクチャ（場合によっては異なる OS）のイメージを 1 つのイメージとして扱えるようにまとめたもの**です。マルチプラットフォームイメージであれば、例えば x86_64 と Arm64 のような異なる CPU アーキテクチャの環境からでも、同じイメージを指定したときに適切なアーキテクチャのイメージが自動で選択されて利用できます。**利用者はアーキテクチャを意識することなく、同じイメージを使うことができる**ため、運用が簡単になります。Docker Hub で提供されている公式イメージにも、マルチプラットフォームイメージとして提供されているイメージが多くあります。

以下は、Docker 公式のマルチプラットフォームイメージの解説です。

@[card](https://docs.docker.com/build/building/multi-platform/)

また、以下のぽよメモ様の記事がとてもわかりやすいのでこちらも参考にしてください。[^1]

@[card](https://poyo.hatenablog.jp/entry/2021/09/25/225329)

[^1]: 今回の記事は多くの部分をぽよメモ様の記事を参考にさせていただいています、ありがとうございます🙇‍♂️

## GitHub Actions でマルチプラットフォームイメージをビルドする従来の方法（QEMU）

GitHub Actions でマルチプラットフォームイメージをビルドする方法は、以下の Docker 公式サイトの記事で解説されています。

@[card](https://docs.docker.com/build/ci/github-actions/multi-platform/)

これまでは GitHub Actions には Arm64 ランナーがなかった[^2]ため、[QEMU](https://www.qemu.org/) というプロセッサエミュレータを使ってマルチプラットフォームイメージをビルドする方法が一般的でした。

[^2]: Arm64 はこれまでも Apple silicon (M1) macOS ランナーがあったのではと思われる方がいるかもしれませんが、Apple silicon macOS ランナーは nested virtualization をサポートしていないため Docker コマンドが使えませんでした（[参考](https://github.com/orgs/community/discussions/69211#discussioncomment-7197681)）

例として、x86_64 と Arm64 のマルチプラットフォームイメージをビルドするワークフローは以下のようになります。

```yaml
name: Build and push multi-platform image

on: push

env:
  REGISTRY_IMAGE: miyajan/multi-platform-example

jobs:
  build:
    runs-on: ubuntu-latest
    strategy:
      fail-fast: false
      matrix:
        platform:
          - linux/amd64
          - linux/arm64
    steps:
      - name: Prepare
        run: |
          platform=${{ matrix.platform }}
          echo "PLATFORM_PAIR=${platform//\//-}" >> $GITHUB_ENV

      - name: Checkout
        uses: actions/checkout@v4

      - name: Docker meta
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ${{ env.REGISTRY_IMAGE }}

      - name: Set up QEMU
        uses: docker/setup-qemu-action@v3
        with:
          platforms: ${{ matrix.platform }}

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Login to Docker Hub
        uses: docker/login-action@v3
        with:
          username: ${{ secrets.DOCKERHUB_USERNAME }}
          password: ${{ secrets.DOCKERHUB_TOKEN }}

      - name: Build and push by digest
        id: build
        uses: docker/build-push-action@v5
        with:
          context: .
          platforms: ${{ matrix.platform }}
          labels: ${{ steps.meta.outputs.labels }}
          outputs: type=image,name=${{ env.REGISTRY_IMAGE }},push-by-digest=true,name-canonical=true,push=true

      - name: Export digest
        run: |
          mkdir -p /tmp/digests
          digest="${{ steps.build.outputs.digest }}"
          touch "/tmp/digests/${digest#sha256:}"

      - name: Upload digest
        uses: actions/upload-artifact@v4
        with:
          name: digests-${{ env.PLATFORM_PAIR }}
          path: /tmp/digests/*
          if-no-files-found: error
          retention-days: 1

  merge:
    runs-on: ubuntu-latest
    needs:
      - build
    steps:
      - name: Download digests
        uses: actions/download-artifact@v4
        with:
          path: /tmp/digests
          pattern: digests-*
          merge-multiple: true

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Docker meta
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ${{ env.REGISTRY_IMAGE }}

      - name: Login to Docker Hub
        uses: docker/login-action@v3
        with:
          username: ${{ secrets.DOCKERHUB_USERNAME }}
          password: ${{ secrets.DOCKERHUB_TOKEN }}

      - name: Create manifest list and push
        working-directory: /tmp/digests
        run: |
          docker buildx imagetools create $(jq -cr '.tags | map("-t " + .) | join(" ")' <<< "$DOCKER_METADATA_OUTPUT_JSON") \
            $(printf '${{ env.REGISTRY_IMAGE }}@sha256:%s ' *)

      - name: Inspect image
        run: |
          docker buildx imagetools inspect ${{ env.REGISTRY_IMAGE }}:${{ steps.meta.outputs.version }}
```

Dockerfile はぽよメモ様の記事を参考に以下のように書きました。

```Dockerfile
FROM golang:1.22.3 as builder

ARG VERSION=latest

RUN go install github.com/Code-Hex/Neo-cowsay/cmd/v2/cowsay@${VERSION}

FROM gcr.io/distroless/static

COPY --from=builder /go/bin/cowsay /usr/bin/cowsay

ENTRYPOINT ["/usr/bin/cowsay"]
```

ワークフローを実行すると、以下の画像のように、無事 Docker Hub 上にマルチプラットフォームイメージがプッシュされたことを確認できました。

![マルチプラットフォームイメージ](/images/build-multi-platform-image-with-arm64-runner/docker-hub.png)

ここで問題となるのがジョブの実行時間です。以下の画像のように、**x86_64 イメージのビルドジョブの実行時間が 1 分を切るのに対し、Arm64 イメージのビルドジョブの実行時間は 3 分を超えています**。

![ビルド時間](/images/build-multi-platform-image-with-arm64-runner/workflow-graph.png)

これは、**QEMU によるエミュレーションのオーバーヘッドが原因**です。今回は小さいイメージなので 3 分で済んでいますが、より大きなイメージのビルドで 3 倍以上の時間がかかるとなるとかなり厳しい事態が想像できます。

## Arm 64 ランナーを使いマルチプラットフォームイメージをビルドする方法

QEMU を使わずに、今回新しく追加された Arm64 ランナーを使うことで、このオーバーヘッドを削減することができるようになるはずです。

### Arm64 ランナーの追加

Arm64 ランナーは現時点では [larger runner](https://docs.github.com/en/actions/using-github-hosted-runners/about-larger-runners/about-larger-runners) 扱いなので、Arm64 ランナーを使えるようにするために Organization の設定の `Actions` → `Runners` からランナーを追加します。以下の画像の、`New GitHub-hosted runner` から追加できます。

![新しいランナーの追加](/images/build-multi-platform-image-with-arm64-runner/new-github-hosted-runner.png)

今回は、`ubuntu-22.04-2core-arm64` という名前で Arm64、Ubuntu 22.04、CPU 2 コアのランナーを追加します。

![Arm64 ランナーの作成](/images/build-multi-platform-image-with-arm64-runner/create-arm64-runner.png)

### ワークフローの修正

先ほどのワークフローを、作成した Arm64 ランナーを使うように修正します。主に `matrix` 部分を修正して、プラットフォームごとに異なるランナーを使うようにし、QEMU を使わないようにしています。

```yaml
name: Build and push multi-platform image

on: push

env:
  REGISTRY_IMAGE: miyajan/multi-platform-example

jobs:
  build:
    strategy:
      fail-fast: false
      matrix:
        include:
          - platform: linux/amd64
            runner: ubuntu-latest
          - platform: linux/arm64
            runner: ubuntu-22.04-2core-arm64
    runs-on: ${{ matrix.runner }}
    steps:
      - name: Prepare
        run: |
          platform=${{ matrix.platform }}
          echo "PLATFORM_PAIR=${platform//\//-}" >> $GITHUB_ENV

      - name: Checkout
        uses: actions/checkout@v4

      - name: Docker meta
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ${{ env.REGISTRY_IMAGE }}

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Login to Docker Hub
        uses: docker/login-action@v3
        with:
          username: ${{ secrets.DOCKERHUB_USERNAME }}
          password: ${{ secrets.DOCKERHUB_TOKEN }}

      - name: Build and push by digest
        id: build
        uses: docker/build-push-action@v5
        with:
          context: .
          platforms: ${{ matrix.platform }}
          labels: ${{ steps.meta.outputs.labels }}
          outputs: type=image,name=${{ env.REGISTRY_IMAGE }},push-by-digest=true,name-canonical=true,push=true

      - name: Export digest
        run: |
          mkdir -p /tmp/digests
          digest="${{ steps.build.outputs.digest }}"
          touch "/tmp/digests/${digest#sha256:}"

      - name: Upload digest
        uses: actions/upload-artifact@v4
        with:
          name: digests-${{ env.PLATFORM_PAIR }}
          path: /tmp/digests/*
          if-no-files-found: error
          retention-days: 1

  merge:
    runs-on: ubuntu-latest
    needs:
      - build
    steps:
      - name: Download digests
        uses: actions/download-artifact@v4
        with:
          path: /tmp/digests
          pattern: digests-*
          merge-multiple: true

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Docker meta
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ${{ env.REGISTRY_IMAGE }}

      - name: Login to Docker Hub
        uses: docker/login-action@v3
        with:
          username: ${{ secrets.DOCKERHUB_USERNAME }}
          password: ${{ secrets.DOCKERHUB_TOKEN }}

      - name: Create manifest list and push
        working-directory: /tmp/digests
        run: |
          docker buildx imagetools create $(jq -cr '.tags | map("-t " + .) | join(" ")' <<< "$DOCKER_METADATA_OUTPUT_JSON") \
            $(printf '${{ env.REGISTRY_IMAGE }}@sha256:%s ' *)

      - name: Inspect image
        run: |
          docker buildx imagetools inspect ${{ env.REGISTRY_IMAGE }}:${{ steps.meta.outputs.version }}
```

結果、以下の画像のように、**Arm64 イメージのビルドジョブも 1 分を切り、x86_64 イメージのビルドジョブと同等の時間になりました**。

![Arm64 ランナーを使ったビルド時間](/images/build-multi-platform-image-with-arm64-runner/workflow-graph-with-arm64.png)

## まとめ

今回は、GitHub Actions に新しく追加された Arm64 ランナーを使って Docker のマルチプラットフォームイメージをビルドしてみました。**Arm64 ランナーを使うことで、QEMU によるオーバーヘッドを削減し、ビルド時間を短縮することができました**。

これまでも Arm64 のマシンを別途用意してリモートビルダーとして使うことで同等のことはできたはずではありますが、専用のマシンを運用し続けるのはややハードルが高いところがありました。**GitHub Actions に Arm64 ランナーが追加されたことにより、より手軽にマルチプラットフォームイメージをビルドできるようになった**のではないでしょうか。

正直、自分は本格的にマルチプラットフォームイメージを運用したことがないため、この記事にもツッコミどころや改善点があるかもしれません。もし何かありましたら、ぜひコメントなどで教えていただけると嬉しいです。
