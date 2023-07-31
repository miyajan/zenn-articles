---
title: "GitHub Actions の OpenID Connect サポートについて"
emoji: "🆔"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["githubactions", "oidc", "aws"]
published: true
---

この記事は [GitHub Actions Advent Calendar 2021](https://qiita.com/advent-calendar/2021/github-actions) の 3 日目の記事です。

2021/10/27 に GitHub Actions の OpenID Connect (OIDC) サポートが[正式にアナウンス](https://github.blog/changelog/2021-10-27-github-actions-secure-cloud-deployments-with-openid-connect/)されました。

この機能を一通り触ってみて気づいたことをまとめます。

## 概要

これまで、GitHub Actions のワークフロー実行中に AWS や GCP といったクラウドプロバイダにアクセスする必要がある場合、クラウドプロバイダ側でクレデンシャルを発行して GitHub 側にシークレットとして保存するのが一般的でした。

しかし、GitHub に長時間有効なクラウドプロバイダのシークレットを保存すると、例えば退職者が発生したときにシークレットを更新する作業が必要になるなど、面倒な作業が発生してしまいます。また、シークレットの漏洩リスクについても考慮が必要になります。[^1]

[^1]: これまでも GitHub Actions では Environments という機能でシークレットの利用を保護することが可能ではありました

これに対して、今回サポートされた OIDC による認証を使うと、以下の図のようなフローで GitHub Actions のワークフローからクラウドプロバイダへの認証が行えるようになります。

![認証フロー](/images/github-actions-support-openid-connect/auth-flow.png)

これにより、以下のようなメリットがあります。

- クラウドプロバイダ側でシークレットを発行して GitHub 側に保存する必要がないので、漏洩リスクが減る
- OIDC により発行されるトークンが有効なのは短時間なので、退職者発生時などのシークレットローテーションが必要なくなる
- クラウドプロバイダ側で OIDC による認証を許可する条件を柔軟に設定できる

## 例: AWS に OIDC で認証する

説明だけだとよくわからないと思うので、実際に OIDC の仕組みを使って AWS で認証してみます。

### 前提条件

以下が用意されている前提で書きます。

- GitHub リポジトリ
  - この例では `miyajan/github-actions-oidc-test` とします
- AWS アカウント

### AWS の設定

#### ID プロバイダの設定

![ID プロバイダの設定](/images/github-actions-support-openid-connect/setup-id-provider.png)

「IAM」→「ID プロバイダ」から、以下の設定でプロバイダを追加します。

- プロバイダのタイプ: OpenID Connect
- プロバイダの URL: https://token.actions.githubusercontent.com
- 対象者: sts.amazonaws.com

これで GitHub が提供する OIDC プロバイダの情報を AWS に登録できました。

#### ロールの設定

まず、適当なロールを作成します。この例では、GitHubActionsOIDCTest というロール名にします。

そして、ロールの信頼関係のポリシーを以下のように設定します。

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::<AWSアカウントID>:oidc-provider/token.actions.githubusercontent.com"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "token.actions.githubusercontent.com:sub": "repo:miyajan/github-actions-oidc-test:ref:refs/heads/main"
        }
      }
    }
  ]
}
```

↑のポリシーで、`miyajan/github-actions-oidc-test` リポジトリの `main` ブランチで実行された GitHub Actions ワークフローでのみ、このロールの利用を許可する設定になります。[^2]

[^2]: この記事では特に AWS の権限が必要な操作をしませんが、実際の用途では必要な権限を付与したポリシーをロールに追加する必要があります

#### おまけ：Terraform

ここまでに作ったリソースを Terraform 化すると以下のようになります。普段 Terraform を使う人は参考にしてください。

```hcl
resource "aws_iam_openid_connect_provider" "github_actions" {
  url = "https://token.actions.githubusercontent.com"

  client_id_list = ["sts.amazonaws.com"]

  # dummy value
  thumbprint_list = ["0123456789012345678901234567890123456789"]
}

resource "aws_iam_role" "github_actions_oidc_test" {
  name = "GitHubActionsOIDCTest"

  assume_role_policy = jsonencode({
    "Version" : "2012-10-17",
    "Statement" : [
      {
        "Effect" : "Allow",
        "Principal" : {
          "Federated" : aws_iam_openid_connect_provider.github_actions.arn
        },
        "Action" : "sts:AssumeRoleWithWebIdentity",
        "Condition" : {
          "StringEquals" : {
            "token.actions.githubusercontent.com:sub" : "repo:miyajan/github-actions-oidc-test:ref:refs/heads/main"
          },
        }
      }
    ]
  })
}
```

:::message
2023/07/31 追記
GitHub Actions の OIDC と AWS の連携が改善され、`thumprint_list` の値は使われなくなりました。ただし、`thumbprint_list` の設定は必須なため、上の例ではダミーの値を指定しています。詳細は、[GitHub Actions - OIDC integration with AWS no longer requires pinning of intermediate TLS certificates - The GitHub Blog](https://github.blog/changelog/2023-07-13-github-actions-oidc-integration-with-aws-no-longer-requires-pinning-of-intermediate-tls-certificates/) を参照してください。

~~2022/08/26 追記~~
~~記事公開時は `thumbprint_list` の値をベタ書きしていましたが、[Terraform だけを使って GitHub Actions OIDC ID プロパイダの thumbprint を計算する方法](https://zenn.dev/yukin01/articles/github-actions-oidc-provider-terraform)の記事とコメント欄を参考に、Terraform の [http](https://registry.terraform.io/providers/hashicorp/http/latest/docs/data-sources/http) と [tls_certificate](https://registry.terraform.io/providers/hashicorp/tls/latest/docs/data-sources/certificate) の Data Source を使って自動で取得するように修正しました。~~
:::

### GitHub Actions の設定

GitHub Actions のワークフローの設定は以下のようになります。Secrets の `AWS_ACCOUNT_ID` に使用する AWS アカウントの ID が入ってる前提です。

```yaml
name: OIDC Test

on: push

permissions:
  id-token: write
  contents: read # actions/checkout のために必要

jobs:
  get-caller-identity:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v2
      - uses: aws-actions/configure-aws-credentials@v1
        with:
          role-to-assume: arn:aws:iam::${{ secrets.AWS_ACCOUNT_ID }}:role/GitHubActionsOIDCTest
          aws-region: ap-northeast-1
      - run: aws sts get-caller-identity
```

`permissions` の `id-token: write` は GitHub の OIDC プロバイダから発行されるトークンを利用するために必要です。そして、`permissions` を書くと他の `contents: read` のようなデフォルトで有効になってる権限も無効になってしまうため、他の必要な権限も明示的に書く必要があります。

`aws-actions/configure-aws-credentials` は AWS 公式のアクションで、他に認証情報を渡さなければ自動で OIDC トークンを使ってくれます。簡単ですね。

ワークフローが正しく動けば、`aws sts get-caller-identity` で以下のように作成したロールで認証できたことを確認できます。[^3]

```json
{
    "UserId": "***:GitHubActions",
    "Account": "***",
    "Arn": "arn:aws:sts::***:assumed-role/GitHubActionsOIDCTest/GitHubActions"
}
```

[^3]: `GitHubActions` は `aws-actions/configure-aws-credentials` がデフォルトで設定するセッション名で、パラメータで変更できます

## GitHub Actions の OIDC トークンの `sub` にはなにが入るのか？

先ほどの例で、ロールの信頼関係に以下の `Condition` を設定しました。

```json
"StringEquals": {
  "token.actions.githubusercontent.com:sub": "repo:miyajan/github-actions-oidc-test:ref:refs/heads/main"
}
```

これは GitHub の ID プロバイダが発行する OIDC トークンに含まれる `sub` の値を条件にしています。`sub` の値は GitHub 側がトークンを発行するときに GitHub Actions のワークフロー実行のきっかけとなったイベントに対応して決まるようです。`sub` の値がどのように決まるか詳細が公式ドキュメントに見当たらないため正確な仕様はわかりませんが、自分が試した範囲では以下のようになっていました。

| ワークフローをトリガーするイベント | `sub` |
| --- | --- |
| ブランチを `push` | repo:<owner>/<repo>:ref:refs/heads/<branch> |
| タグを `push` | repo:<owner>/<repo>:ref:refs/tags/<tag> |
| `pull_request` | repo:<owner>/<repo>:pull_request |
| `workflow_dispatch` | repo:<owner>/<repo>:ref:refs/heads/<branch> |
| `schedule` | repo:<owner>/<repo>:ref:refs/heads/<branch> |
| `environment` を指定して `push` or `pull_request` or `workflow_dispatch` or `schedule` | repo:<owner>/<repo>:environment:<environment> |

なので、この `sub` の値を使うことでクラウドプロバイダ側で認証を許可する条件を柔軟に設定できます。

例えば AWS では、上の例では main ブランチでワークフローが実行されたときのみ認証を許可していますが、`sub` キーの値を変えることで、特定のタグでワークフローが実行されたときのみ、`pull_request` イベントでワークフローが実行されたときのみ、特定の environment を使うワークフローでのみ、といった様々な条件で認証を許可するか制限することができます。

また、AWS では `StringLike` を使うとワイルドカードが使えるので、以下のようにすると特定のリポジトリのすべてのワークフローで認証を許可できます。

```json
"StringLike": {
  "token.actions.githubusercontent.com:sub": "repo:miyajan/github-actions-oidc-test:*"
}
```

逆に、クラウドプロバイダ側でアクセス許可の条件に `sub` の値を設定しなかった場合、他人のリポジトリから認証されてしまう危険があるので、必ず設定するようにしましょう。

ちなみに `sub` は OpenID Connect でユーザーの一意な識別子を表すパラメータです。OIDC トークンには sub 以外にも[さまざまな情報](https://docs.github.com/en/actions/deployment/security-hardening-your-deployments/about-security-hardening-with-openid-connect#understanding-the-oidc-token)が含まれるのでそちらをアクセス許可の条件にすることも考えられますが、GitHub の公式ドキュメントには `sub` を条件にする例しか書かれていません。おそらく、OpenID Connect としては `sub` で許可する権限が決まるのが一般的なのでそうしているのではと思われます。

## OpenID Connect についていろいろ

ここからは本題と少し逸れますが、OIDC について気になりそうなところをいろいろ書いていきます。今回紹介した仕組みを使う上でこの知識は必須というわけではないので、読み飛ばしてもらって問題ありません。また、筆者は OIDC やその関連技術について詳しいわけではないので、ツッコミどころなどあったら教えていただけると助かります。

### OpenID Connect ってなに？

OAuth という認可の仕組みを拡張して、認証を行える仕組みを作ったのが OpenID Connect です。[^4]

[^4]: 認可は権限の移譲を行うことであり、認証は相手が誰かを確認することです

OpenID Connect の認証のためのトークンを発行するのが ID プロバイダです。今回では、GitHub が OIDC の ID プロバイダを提供しており、GitHub Actions のワークフローは ID プロバイダからトークンを発行してもらって、クラウドプロバイダに送ります。クラウドプロバイダ側は ID プロバイダを信頼する設定を元にトークンを検証して、問題なければ認証成功とします。

### GitHub Actions のワークフローはどうやって GitHub の ID プロバイダからトークンを取得してるの？

[@actions/core](https://www.npmjs.com/package/@actions/core) という npm パッケージに `getIDToken()` という OIDC トークンを取得するメソッドがあり、例で紹介した `aws-actions/configure-aws-credentials` アクションはこのメソッドを内部的に呼び出しています。[ソースコード](https://github.com/actions/toolkit/blob/4df5abb3eef67a5170035145f1b8c102f5e12ee3/packages/core/src/oidc-utils.ts)を読むとどうやってトークンを取得しているかがわかります。

ワークフローには `ACTIONS_ID_TOKEN_REQUEST_URL` という環境変数で ID プロバイダからトークンを取得するための URL が与えられ、`ACTIONS_ID_TOKEN_REQUEST_TOKEN` という環境変数で与えられたトークンを送ることによって OIDC トークンを取得します。

### クラウドプロバイダは OIDC トークンをどうやって検証してるの？

OIDC のトークンは、以下のような JSON Web Token (JWT) 形式になっています。（ヘッダ、ペイロード、署名はそれぞれ Base64 でエンコードされています）

```
(ヘッダ).(ペイロード).(署名)
```

これらの情報と ID プロバイダの公開鍵を使って署名の検証を行います。

まず、ヘッダを base64 デコードして見ると以下のようになっています。

```json
{
  "typ": "JWT",
  "alg": "RS256",
  "x5t": "（省略）",
  "kid": "（省略）"
}
```

`alg` が署名で使われるアルゴリズムです。`RS256` は暗号化に [RSA](https://ja.wikipedia.org/wiki/RSA%E6%9A%97%E5%8F%B7) という公開鍵暗号を使い、ハッシュの生成に SHA-256 を使います。

署名生成や検証方法の詳細は省きますが、ID プロバイダは `(ヘッダ).(ペイロード)` のハッシュを秘密鍵を使って暗号化したものを署名として追加し、トークンを受け取った側は署名を ID プロバイダの公開鍵を使って復号化したものが `(ヘッダ).(ペイロード)` のハッシュと一致することを検証します。

ID プロバイダの公開鍵を取得するためのエンドポイントは [OpenID Connect Discovery 1.0](https://openid.net/specs/openid-connect-discovery-1_0.html) という仕様で決まっており、ID プロバイダの `/.well-known/openid-configuration` というパスで取得できます。GitHub の場合、以下の URL で取得されています。

[https://token.actions.githubusercontent.com/.well-known/openid-configuration](https://token.actions.githubusercontent.com/.well-known/openid-configuration)

↑のレスポンスの JSON の `jwks_uri` から公開鍵を JSON Web Key (JWK) という形式で取得できます。 クライアントプロバイダ側はこれを使って検証します。

ちなみに、OIDC トークンに含まれるペイロードは以下のようになっています。

```json
{
  "jti": "（省略）",
  "sub": "repo:miyajan/github-actions-oidc-test:ref:refs/heads/main",
  "aud": "https://github.com/miyajan",
  "ref": "refs/heads/main",
  "sha": "（省略）",
  "repository": "miyajan/github-actions-oidc-test",
  "repository_owner": "miyajan",
  "run_id": "（省略）",
  "run_number": "（省略）",
  "run_attempt": "（省略）",
  "actor": "miyajan",
  "workflow": "OIDC Test",
  "head_ref": "",
  "base_ref": "",
  "event_name": "push",
  "ref_type": "branch",
  "job_workflow_ref": "miyajan/github-actions-oidc-test/.github/workflows/test.yml@refs/heads/main",
  "iss": "https://token.actions.githubusercontent.com",
  "nbf": 1638357172,
  "exp": 1638358072,
  "iat": 1638357772
}
```

`nbf` の値以前の時間ではこのトークンは有効ではなく、`exp` の値の時間でトークンの有効期限が切れます。なので、トークンを受け取った側は署名の検証だけではなく、こういった検証も行います。

### GitHub Actions で OIDC トークンのヘッダーやペイロードはどうやったら確認できる？

単純に OIDC トークンを "." で分割して、それぞれ base64 デコードするだけヘッダーやペイロードの中身を確認できます。自分は以下のような `curl` と `jq` を使ったワンライナーで確認しました。

```shell
curl -s -H "Authorization: bearer ${ACTIONS_ID_TOKEN_REQUEST_TOKEN}" "${ACTIONS_ID_TOKEN_REQUEST_URL}" | jq '.value | split(".") | .[0],.[1] | @base64d | fromjson'
```

### AWS の ID プロバイダの設定にあったサムプリントって何？

:::message
2023/07/31 追記

上でも書いた通り、GitHub Actions の OIDC と AWS の連携が改善され、ID プロバイダの設定にあるサムプリントの値は使われなくなりました。詳細は、[GitHub Actions - OIDC integration with AWS no longer requires pinning of intermediate TLS certificates - The GitHub Blog](https://github.blog/changelog/2023-07-13-github-actions-oidc-integration-with-aws-no-longer-requires-pinning-of-intermediate-tls-certificates/) を参照してください。
:::

~~例にあった通り、AWS だと ID プロバイダの設定画面でサムプリントが表示されます。これは、ID プロバイダの公開鍵証明書に署名した最上位の中間認証局の証明書のフィンガープリントから生成される値です。~~

~~https://qiita.com/minamijoyo/items/eac99e4b1ca0926c4310~~

~~↑の記事がサムプリントの計算方法の参考になります。~~

~~[AWS のドキュメント](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles_providers_create_oidc_verify-thumbprint.html)によると、サムプリントによってその中間認証局が発行した同じ DNS 名の証明書を信頼するとのことなので、ID プロバイダが公開鍵証明書を更新しても対応をする必要がなくなるということのようです。ただし、中間認証局の証明書の期限が 2028/10/22 となっているので、そのタイミングでは更新が必要になると思われます。~~

## まとめ

GitHub Actions の OIDC サポートは、手軽に使えますし、シークレット管理の悩みを軽減してくれますし、とても嬉しいですね。今後は GitHub Actions からクラウドプロバイダへ認証する場合、GitHub に長時間有効なシークレットを保存することは避け、OIDC の仕組みを使うのが基本になると思います。
