---
title: "Serverless Framework から AWS SAM に移行した話"
emoji: "☁️"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["serverless", "aws"]
published: true
published_at: 2024-08-18 15:00
publication_name: cybozu_ept
---

:::message
この記事は、[CYBOZU SUMMER BLOG FES '24](https://cybozu.github.io/summer-blog-fes-2024/) (生産性向上 セイサンシャインビーチ Stage) DAY 7 の記事です。
:::

## 背景

生産性向上チームでは、AWS Lambda のデプロイのために Serverless Framework というツールを使っていました。

@[card](https://www.serverless.com/)

しかし、2023 年の 10 月に Serverless Framework が V.4 になると有償化されることがアナウンスされました。

@[card](https://www.serverless.com/blog/serverless-framework-v4-a-new-model)

年間売上が 200 万ドルを超える組織は有償化の対象となり、Serverless Framework Dashboard 上の Service Instances（`serverless.yml` 上の `service`, `stage`, `region` パラメータの組み合わせ数）, Traces, Metrics の数に応じたクレジットを購入する方式となります。最新の詳細は公式ページをご参照ください。

@[card](https://www.serverless.com/pricing)

V.3 までは無料のままですが、V.3 のセキュリティ修正や重大なバグ修正のサポートは 2024 年いっぱいまでとなります。

私たちにとってはそこまで高額にはならなさそうだったので、お金を払って V.4 を使い続けるという選択肢もありました。しかし、

- （当時）Serverless Framework やその周辺プラグインがあまりメンテされていないと感じる場面がいくつかあり、今後に心配があった
- AWS 公式が提供するツールが Serverless Framework を導入した当時より充実してきている

といった理由から他ツールへ移行することを検討し始めました。

## 移行先の検討

まず、Lambda のデプロイのためには以下のような作業が必要です。

1. Lambda 関数が依存するリソースのデプロイ
    - 例: VPC、セキュリティグループ、サブネット、など 
2. Lambda 関数のビルド
    - 例: Node.js なら esbuild でビルドする、など 
3. パッケージ生成、アップロード
    - 例: Zip 化して S3 にアップロードする、Docker イメージを ECR にアップロードする、など
4. Lambda 関数のデプロイ
5. Lambda 関数に依存するリソースのデプロイ
    - 例: ELB のターゲットグループ、など

私たちはこれまで、1.（とそれ以外の Lambda 関係以外のリソースのデプロイ）を Terraform で行っており、Serverless Framework はそれ以外の 2. 〜 5. の作業を行ってくれていました。

今回、2. 〜 5. をすべてやってくれるツールとしては、以下を検討しました。

- [AWS SAM (Serverless Application Model)](https://docs.aws.amazon.com/ja_jp/serverless-application-model/latest/developerguide/what-is-sam.html)
- [AWS CDK (Cloud Development Kit)](https://docs.aws.amazon.com/ja_jp/cdk/v2/guide/home.html)

これらは、2 つとも AWS 公式のツールです。AWS SAM は、CloudFormation を拡張したような YAML 設定ファイルを書き、[SAM CLI](https://github.com/aws/aws-sam-cli) を使って Lambda をデプロイできます。AWS CDK は、コードでインフラストラクチャを定義できます。

また、2. 〜 5. をすべて一つのツールでやらないという選択肢もあります。例えば Lambda 関数のビルドとパッケージ生成＆アップロードを自前でスクリプトを書くなどして行い、残りのリソース定義は Terraform で行うような選択肢が考えられます。

様々な検討の結果、大半は AWS SAM に移行することにしました。理由は以下のとおりです。

- AWS 公式のツールであり、新しいランタイムや新機能などに追従しやすいと考えられる
- YAML だけで記述でき、SAM CLI によるオペレーションも理解しやすい
- プログラマティックな処理が求められるケース以外では CDK を使うメリットは薄い、むしろ余計な依存が発生すると判断[^1]

[^1]: 設定ファイルから作成するリソースをプログラマティックに決めたいプロジェクトでは CDK を選択したケースもあります

## 移行作業

移行作業は、大まかに以下の流れで行いました。

1. AWS SAM の設定ファイルを作成する
2. AWS SAM CLI でリソースをデプロイする
    - この時点で Serverless Framework でデプロイしたリソースと AWS SAM でデプロイしたリソースが共存する
3. 動作確認する
4. Serverless Framework のリソースを削除する
5. Serverless Framework の設定ファイルを削除する

我々の場合は同じ Lambda が二重に存在しても問題ないケースだけだったのでこの方法をとりましたが、同時実行されては困る Lambda が存在するケースなどでは他の方法を検討する必要があります。[^2]

[^2]: AWS SAM には既存リソースをインポートする方法があるのですが、調査した感じでは予想以上に扱いづらくあまりおすすめしません（[参考](https://dev.classmethod.jp/articles/aws-sam-template-import-resources/)）

ここからは、詳細を書きます。

### AWS SAM の設定ファイルを作成する

SAM の設定ファイルには、`template.yaml` と `samconfig.toml` があります。`template.yaml` は CloudFormation のように書けるテンプレートファイルで、`samconfig.toml` は SAM CLI の設定ファイルです。

#### `template.yaml`

`template.yaml` の例は、以下のようになります。[^3]

[^3]: `sam init` コマンドで TypeScript の Hello World テンプレートを作成した場合の `template.yaml` を一部改変したものです

```yaml
AWSTemplateFormatVersion: '2010-09-09'
Transform: AWS::Serverless-2016-10-31
Description: >
  sam-app

  Sample SAM Template for sam-app
  
# More info about Globals: https://github.com/awslabs/serverless-application-model/blob/master/docs/globals.rst
Globals:
  Function:
    Timeout: 3

Resources:
  HelloWorldFunction:
    Type: AWS::Serverless::Function # More info about Function Resource: https://github.com/awslabs/serverless-application-model/blob/master/versions/2016-10-31.md#awsserverlessfunction
    Properties:
      CodeUri: hello-world/ # template.yaml と同じところに hello-world ディレクトリがあり、その下に Lambda 関数のコードがあるという前提
      Handler: app.lambdaHandler
      Runtime: nodejs20.x
      Architectures:
        - x86_64
      Events:
        HelloWorld:
          Type: Api # More info about API Event Source: https://github.com/awslabs/serverless-application-model/blob/master/versions/2016-10-31.md#api
          Properties:
            Path: /hello
            Method: get
    Metadata: # Manage esbuild properties
      BuildMethod: esbuild
      BuildProperties:
        Minify: true
        Target: "es2020"
        Sourcemap: true
        EntryPoints: 
        - app.ts # hello-world ディレクトリの下に app.ts があるという前提
```

CloudFormation を触ったことがある人なら理解しやすいと思いますし、触ったことがない人でも Serverless Framework のようなツールを使ったことがあれば一つ一つの設定は理解しやすいと思います。ただ、Serverless Framework では設定ファイルに書かなくても自動で作成されていたリソース（IAM ロールや CloudWatch Logs のロググループなど）は自動で作られないため、上の例では省略してますがそれらを明示的にリソースとして定義する必要があります。

シークレットは、Secrets Manager から動的に読み込む仕組みを利用できます。

@[card](https://docs.aws.amazon.com/ja_jp/AWSCloudFormation/latest/UserGuide/dynamic-references.html)

それ以外だとパラメータを渡す `Parameters` という設定項目があります。ただし、パラメータの Default は最初の 1 回しか読み込まれず、Default を変更しても反映されないので注意が必要です。[^4]

[^4]: Default を使うのではなく `samconfig.toml` の `parameter_overrides` でデフォルト値を渡すようにしておくという回避策はあります 

@[card](https://dev.classmethod.jp/articles/cloudformation-parameter-default/)

作成するリソース名を Serverless Framework のリソースと同じにしてしまうと、共存期間中に名前が重複禁止になってる種類のリソースはエラーになってしまうため、リソース名は可能であれば変更するのが無難です。

また、`sam validate` コマンドを実行することで `template.yaml` の構文チェックを行えます。`-lint` オプションをつけると `cfn-lint` という CloudFormation の linter もかけてくれるので、後述する `samconfig.toml` で設定するのがおすすめです。

@[card](https://docs.aws.amazon.com/serverless-application-model/latest/developerguide/sam-cli-command-reference-sam-validate.html)

#### `samconfig.toml`

`samconfig.toml` の例は、以下のようになります。

```yaml
version = 0.1

[default]
[default.global.parameters]
stack_name = "sam-app"

[default.build.parameters]
cached = true
parallel = true

[default.validate.parameters]
lint = true

[default.deploy.parameters]
capabilities = "CAPABILITY_NAMED_IAM"
confirm_changeset = true 
resolve_s3 = true
```

`default.xxx.parameters` セクションの設定は、SAM CLI の `sam xxx` にオプションとして渡すパラメータに対応します。例えば、`default.build.parameters` セクションであれば `sam build` コマンドにデフォルトで渡すオプションを設定しています。`default.global.parameters` はすべての `sam` コマンドにデフォルトで渡すオプションになります。

@[card](https://docs.aws.amazon.com/ja_jp/serverless-application-model/latest/developerguide/serverless-sam-cli-command-reference.html)

Lambda と一緒に IAM リソースをデプロイする場合は、`deploy` に `capabilities` オプションを渡す必要があるので注意が必要です。

@[card](https://qiita.com/YAMAO_456/items/8104c325c24644c0baeb)

### AWS SAM CLI でリソースをデプロイする

SAM CLI のインストールは公式ドキュメントを参照してください。[^5]

[^5]: Homebrew は公式サポートは終わったようですが、コミュニティサポートはあるようです（https://dev.classmethod.jp/articles/change-of-support-model-of-sam-cli-installation-experience-of-homebrew/）

@[card](https://docs.aws.amazon.com/ja_jp/serverless-application-model/latest/developerguide/install-sam-cli.html)

`sam build` コマンドを実行すると、Lambda 関数のビルド（例: Node.js なら esbuild）が設定どおりに動作するか確認でき、まだ後続のデプロイなどに必要なファイルが `.aws-sam` 下に生成されます。

`sam deploy` コマンドを実行すると、作成される CloudFormation スタックのチェンジセットが表示され、そのままデプロイするか確認が求められます。CLI 上の変更差分はリソース名までで詳細が表示されないのですが、マネジメントコンソール上から CloudFormation の対象のスタックの変更セットを確認するとリソースのプロパティレベルでの差分を確認できます。問題なさそうであればそのままデプロイします。

:::message
最初のデプロイでエラーになった場合、`ROLLBACK_COMPLETE` ステータスになったスタックを手動で削除しないと再デプロイできないので注意が必要です。
:::

今後 Lambda 関数のソースコードを変更して再度デプロイするときは `sam build` を再実行しないと手元の以前のビルドのキャッシュが使われるようなので、デプロイするときは `sam build` を必ず実行してから `sam deploy` を実行するようにしましょう。 

### 動作確認する

主に以下のようなことを確認します。

- CloudFormation のスタックが `CREATE_COMPLETE` （更新してるなら `UPDATE_COMPLETE`）になってること
- Lambda が正常に動作すること
- Lambda の実行ログが指定したロググループに書き出されていること
- 関連リソースがある場合、それらの動作確認
  - 例: CloudWatch Alarm + Chatbot とか

### Serverless Framework のリソースを削除する

`npx sls remove` コマンドを実行すると Serverless Framework のリソースがすべて削除されます。（コマンドを実行すると特に確認などなく全リソース削除されます）

### Serverless Framework の設定ファイルを削除する

主に以下のようなものを削除します。

- Serverless Framework に関係するパッケージを `npm uninstall`
- `serverless.ts`
- その他、Serverless Framework に関する CI、ドキュメントなどの修正

## Serverless Plugin Datadog の置き換え

うちのチームでは、Lambda のモニタリングをするために [Serverless Plugin Datadog](https://www.serverless.com/plugins/serverless-plugin-datadog) を使ってメトリクスを Datadog に送信しているケースがいくつかありました。このため、AWS SAM で同様のことを実現する方法を考える必要がありました。

AWS SAM では、Datadog のサーバーレスマクロを利用することで Lambda のメトリクスを Datadog に送信することができます。

@[card](https://docs.datadoghq.com/ja/serverless/libraries_integrations/macro/)

AWS SAM の場合、`template.yaml` の `Transform` セクションに以下のような設定を追加することで Lambda レイヤーを追加して Lambda のメトリクスを Datadog に送信できます。（以下の例はランタイムが Node.js の場合で、ランタイムごとに設定方法が異なります）

```yaml

Transform:
  - AWS::Serverless-2016-10-31
  # Datadog によるサーバーレスモニタリングを有効化する
  - Name: DatadogServerless
    Parameters:
      stackName: !Ref "AWS::StackName"
      apiKeySecretArn: "<DATADOG_API_KEY_SECRET_ARN>"
      nodeLayerVersion: "115" # 最新バージョンはここを確認 https://github.com/DataDog/datadog-lambda-js/releases し、vX.Y.Z の Y を使用します
      extensionLayerVersion: "63" # 最新バージョンはここを確認 https://github.com/DataDog/datadog-lambda-extension/releases
      service: "<SERVICE>" # メトリクスの service タグに指定する値
```

Serverless Plugin Datadog とサーバーレスマクロは根本的な仕組みは基本的に同じなのですが、Lambda 関数に渡る Datadog 向けの環境変数（`DD_` で始まる環境変数）が微妙に異なり、Datadog にどういったメトリクスが送信されるかといった設定が変わってしまいます。このあたりはデプロイ後に Serverless Framework のときの Lambda 関数の環境変数と AWS SAM の Lambda 関数の環境変数の差分を見比べて調整しました。

## Serverless Framework から AWS SAM に移行した所感

Serverless Framework から AWS SAM に移行して数ヶ月運用していますが、Serverless Framework のときと比べて大きく不便になった点はなく使えています。

一方で、Lambda 関数の設定にプログラマティックな処理が必要となるときは、シェルでどうにかできなくはないのですが、若干やりづらく感じています。

また、SAM というより CloudFormation にいくつか不便な点がある（差分が確認しづらい、デプロイ失敗時のロールバック処理などが想定外の挙動をすることがある）ため、トライアンドエラーがやりづらく感じるときはあります。

今回は選択しませんでしたが、Lambda 関数のビルドとパッケージ生成＆アップロードを自前スクリプトなどで行い、残りのリソース定義は Terraform で行う選択肢も今後試してみてもよさそうに感じています。

## まとめ

生産性向上チームで Lambda 関数のデプロイを Serverless Framework から AWS SAM に移行した話を書きました。移行自体は特に問題なく行え、現状特に大きな問題なく運用できています。とはいえ、Lambda 関数をデプロイするためには言語ランタイムごとのビルド処理が必要となる点が IaC (Infrastructure as Code) と少し相性が悪く、まだ改善の余地はありそうに感じています。
