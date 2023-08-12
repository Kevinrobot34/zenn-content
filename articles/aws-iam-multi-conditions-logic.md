---
title: "AWS IAM の複数のポリシー・条件の評価ロジック"
emoji: "🧑‍⚖️"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["AWS", "IAM"]
published: false
---

## はじめに

AWS の IAM で最小権限の法則を実現しようとすると、 複数のポリシー・ステートメントを用意したり、その中で複数の条件を書いたりすることがあると思います。
また、明示的にアクセス制限を実現するためにリソースベースポリシーに明示的な拒否を否定の条件演算子と組み合わせたりすることがあると思います。

シンプルな条件であればそれほど悩むことはありませんが、条件が複雑になってくると適切に Statement ブロックを分けたり Condition ブロックを記載する必要が出てくるわけです。特に明示的な拒否と否定条件演算子を組み合わせていると、最終的に何が許可されて何が拒否されるのかわかりにくくなりがちです。

そこで、本記事では IAM におけるポリシー・ステートメント・条件に関するロジックをまとめ、いくつかの具体例と共に紹介します。


## IAM の評価ロジック

### 複数ポリシー・ステートメントの評価

以下の通り、ポリシーやステートメントが複数ある場合にはそれらは論理 `OR` で評価されます。
これは分かりやすいと思いますが、特に複数の明示的拒否のステートメントを扱う場合には改めて認識しておきたい部分です。

> ポリシーに複数のステートメントが含まれている場合、AWS は評価時にステートメント全体に論理 `OR` を適用します。複数のポリシーが 1 つのリクエストに適用される場合、AWS は評価時にすべてのポリシーに論理 `OR` を適用します。

https://docs.aws.amazon.com/ja_jp/IAM/latest/UserGuide/access_policies.html#access_policies-json


明示的拒否は明示的な許可より優先されるわけですが、明示的拒否のステートメントが複数ある場合、それらのどれか一つのステートメントにでもリクエストが該当するとアクセスは拒否されることになります。
明示的拒否の各ステートメントの条件の記載を間違えるとあらゆるアクセスが拒否されたりし得るので注意が必要です。


### 複数条件の評価

以下の通り、Condition ブロックでは、演算子やキーについては AND で、キーに対する Value は OR で評価されます。

> * If your policy statement has multiple condition operators, the condition operators are evaluated using a logical `AND``.
> * If your policy statement has multiple context keys attached to a single condition operator, the context keys are evaluated using a logical `AND`.
> * If a single condition operator includes multiple values for a context key, those values are evaluated using a logical `OR`.
> * If a single negated matching condition operator includes multiple values for a context key, those values are evaluated using a logical `NOR`.
> 
> All context keys in a condition element block must resolve to true to invoke the desired `Allow` or `Deny` effect. 
> ![condition-block](/images/articles/aws-iam-multi-conditions-logic/condition-block.png) <!-- markdown-link-check-disable-line -->



https://docs.aws.amazon.com/ja_jp/IAM/latest/UserGuide/reference_policies_condition-logic-multiple-context-keys-or-values.html



## 具体的な検証

### タグを利用した簡易的な検証

実際のポリシーでは `aws:sourceVpce` や `aws:PrincipalArn` などのコンテキストキーを利用してアクセス経路やアクセス主に応じた許可・拒否を実現したいという場合が多いと思いますが、 IAM policy の評価ロジック（明示的な拒否と否定条件演算子の組み合わせ方、など）を検証するためだけに実際にネットワークの設定を再現したりするのは大変だと思います。

そこで個人的におすすめなのは以下のようにして検証する、というものです。

* 新しく IAM Role を用意し、このロールの Tag と identity based policy は自由に編集できるようにしておく
* S3バケットなどのリソースを用意して、その resource based policy (S3の場合はバケットポリシー) を編集できるようにしておく
* 2つのポリシーを適宜編集して検証する
  * Condition について調査したい場合には `aws:PrincipalTag/tag-key` を Condition に使い、ポリシーやタグを編集したりしながらアクセスが可能か、拒否されるかを調査する

もちろん実際の環境を用意して検証するのが確実ですが、この方法を使えばタグを編集するだけで簡単に様々な条件下でのポリシーの検証をできると思います。


### 具体例

あるS3へのアクセスを明示的に制限するために、

* 基本的には特定の VPC Endpoint からのアクセス以外は Deny
* ある IAM Role に関しては特定の VPC からのアクセス以外は Deny

という2つの条件を満たす明示的拒否のバケットポリシーを作りたいとします。
`Deny` のステートメントと、 `StringNotEquals` といった条件演算子を組み合わせて作ることになりますが、二重否定みたいな感じになるので適切なポリシーを作るのも意外と大変です。

そこで以下の通り3つのタグを用意して、 `aws:PrincipalTag/tag-key` で代わりに実験します。

* `testVpcEndpoint`
* `testVpc`
* `testArn`



#### bucket policy

今回の場合は以下のようなバケットポリシーを作成すれば良いです。

```json
{
    "Version" : "2012-10-17",
    "Statement" : [
      {
        "Principal" : "*",
        "Action" : [
          "s3:GetObject",
          "s3:GetObjectVersion"
        ],
        "Effect" : "Deny",
        "Resource" : "arn:aws:s3:::test-bucket/*",
        "Condition" : {
          "StringNotEquals" : {
            "aws:PrincipalTag/testVpcEndpoint" : [ "vpce-123456780912" ],
            "aws:PrincipalTag/testArn" : "arn:aws:iam::123456789012:role/test-role"
          }
        }
      },
      {
        "Principal" : "*",
        "Action" : [
          "s3:GetObject",
          "s3:GetObjectVersion"
        ],
        "Effect" : "Deny",
        "Resource" : "arn:aws:s3:::test-bucket/*",
        "Condition" : {
          "StringNotEquals" : {
            "aws:PrincipalTag/testVpc" : [ "vpc-12345678" ]
          },
          "StringEquals" : {
            "aws:PrincipalTag/testArn" : "arn:aws:iam::123456789012:role/test-role"
          }
        }
      },
    ]
  }
```

ポイントとしては以下のように `StringNotEquals` に複数のキーがある場合です。
これらは AND として結ばれるので、「`testVpcEndpoint` タグが `vpce-123456780912` ではなく」かつ「`testArn` タグが `arn:aws:iam::123456789012:role/test-role` ではない」場合にDenyされることになります。
つまり「`testVpcEndpoint` タグが `vpce-123456780912` である」もしくは「`testArn` タグが `arn:aws:iam::123456789012:role/test-role` である」場合にのみDenyされない、となります。

```json
"StringNotEquals" : {
  "aws:PrincipalTag/testVpcEndpoint" : [ "vpce-123456780912" ],
  "aws:PrincipalTag/testArn" : "arn:aws:iam::123456789012:role/test-role"
}
```


## References

https://dev.classmethod.jp/articles/s3-bucket-policy-multi-condition/

https://dev.classmethod.jp/articles/devio-2021-iam-evaluation-logic/

https://repost.aws/ja/knowledge-center/iam-policy-tags-deny