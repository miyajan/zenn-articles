---
title: "Selenium 4 の新機能・変更点まとめ"
emoji: "🤖"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Selenium", "Test", "WebDriver"]
published: true
---

2021/10/13 に Selenium 4 の正式リリースがアナウンスされました。
https://www.selenium.dev/blog/2021/announcing-selenium-4/

Selenium 4 の変更点について調べたので、まとめておきます。

## 後方互換性のない変更

基本的には後方互換性ありですが、一部のコードに依存していると修正が必要です。
Selenium 4 へのアップグレード方法のドキュメントが↓にあり、プログラミング言語ごとに修正例が載っているので参考になります。
https://www.selenium.dev/documentation/getting_started/how_to_upgrade_to_selenium_4/

大きなところでは、Capabilities 系のクラスや findElement(s)By* 系のメソッドが使えなくなっています。細かいところでは Java だと Waits や Timeout 系の引数が Duration 型になっているなどいろいろあるので、アップグレードするときはちゃんと上記のドキュメントを読んで影響範囲を把握しましょう。

## 新機能

### Relative Locators

ブラウザ上の他の要素からの相対的な位置（above, below, toLeftOf, toRightOf, near）で要素を指定できるロケーターです。
https://www.selenium.dev/documentation/webdriver/locating_elements/#relative-locators

おもしろくはありますが、結局 UI 変更で位置が変わったらアウトなので実際に使うかは微妙なところですかね。

### BiDi APIs

これまで WebDriver はシンプルなリクエスト/レスポンスモデルでブラウザとやりとりしていましたが、WebDriver BiDirectional Protocol という、クライアントとブラウザが WebSocket で双方向通信するプロトコルができたとのことです。W3C ではまだ Draft ですが、現時点でもいろいろできます。
https://w3c.github.io/webdriver-bidi/

これを使うと、Chrome DevTools Protocol などを経由して、なにかしらのイベントにフックしてなにかさせる系がやりやすくなります。例えば、DOM 変更時や、console.log イベントや、JS エラー発生時などになにか実行したり、ネットワーク通信をインターセプトしたりとかできるようになっているみたいです。
https://www.selenium.dev/documentation/webdriver/bidi_apis/

このあたりの機能はこれまで実現が難しかった自動化を簡単に実現可能にしてくれそうで、とても便利な雰囲気を感じます。

### Selenium Grid

[Zalenium](https://github.com/zalando/zalenium) や [Selenoid](https://github.com/aerokube/selenoid) を参考に、Selenium Grid を作り直したとのことです。
https://www.selenium.dev/documentation/grid/

以前の Grid を構成する要素は Hub と Node だけだったのですが、新しい Grid は構成要素がすごい増えてます。↓にわかりやすい図があるので、理解しやすいです。
https://www.selenium.dev/documentation/grid/components_of_a_grid/

- Router
    - クライアントからのリクエストをルーティングするエントリーポイントになる
- Distributor
    - 新規セッションのリクエストを受け取ってノードを返す
- Node
    - ブラウザを起動するノード（要はマシンとかコンテナとか）
- Session Map
    - セッション ID とノードの関連付けを保存する
- New Session Queue
    - 新規セッションのリクエストを保存する FIFO キュー
- Event Bus
    - Nodes, Distributor, New Session Queue, Session Map 間のコミュニケーションパスとなる

コンポーネントは増えたけど jar は共通のようです。なので、これまでどおり standalone や、hub/node の単位でセットアップすることもできますし、分散構成にもできます。
https://www.selenium.dev/documentation/grid/setting_up_your_own_grid/

Observability は OpenTelemetry に対応してたり、API は GraphQL をサポートしてたりします。
https://www.selenium.dev/documentation/grid/advanced_features/

CLI Options 見てると Router が username と password を持ってるので、認証設定できるようになってそうですね。
https://www.selenium.dev/documentation/grid/configuring_components/cli_options/

Docker コンテナについては Selenium のドキュメントでは触れられておらずですが、引き続き↓のリポジトリにあります。
https://github.com/SeleniumHQ/docker-selenium

BETA になっている Vide recording や Dynamic Grid のあたりが熱い感じがしますね。

## その他

公式アナウンス記事に載ってる以外にもいろいろ改善点はあるようで、applitools の記事がよくまとまっています。
https://applitools.com/blog/selenium-4/

ざっと目についたのを箇条書きにすると、以下のような変更点があるようです。（Selenium IDE は最近の変更なのかよくわかってないので触れません）

- エラーがわかりやすくなった
- 新しい ChromiumDriver は Chrome と Edge 両方に対応
- Firefox ブラウザにアドオンをインストールするメソッドが追加
- フルスクリーンとか新規ウィンドウ、タブの扱いなどがコントロールしやすくなった
- 要素単位のスクリーンショット取得、Firefox でのページ全体のスクリーンショット取得

## まとめ

Selenium 4 のアルファ版は 2 年以上前から出ていたのですが、ようやく正式リリースされました。今回のリリースは安定版かと思っていたのですが、調べてみるとかなり大きな変更が入っていて、Selenium の進化を感じます。あと、[公式サイト](https://www.selenium.dev/)がいつの間にかすごい見やすくなったなと思いました。

新規で E2E テスト自動化を始めるのであれば Selenium 以外の E2E テストに特化したツール（[Cypress](https://github.com/cypress-io/cypress) や [Playwright](https://github.com/microsoft/playwright) など）の検討をおすすめしますが、既存の自動テストをメンテナンスしていくケースなど引き続き Selenium を使う開発者は多いと思うので、今後も Selenium の進化は楽しみですね。
