---
title: "AWS IAM 関係メモ"
emoji: "🔐"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["AWS", "IAM",]
published: true
---

AWS IAM 関連のメモ

## 認証と認可
多くの場合認証と認可は同時に行われるが、本質的には全然違うものであるので、これを意識しておくと分かりやすくなる。

* 認証 (Authentication)
  * 本人性の確認
  * 具体例
    * ID/Passwordの組み合わせで誰が利用しているのかをサーバーが判別する
* 認可 (Authorization)
  * 利用権限の付与
  * 具体例
    * あるユーザーに対してs3のバケットの参照権限を付与する

IAMユーザー等で認証し、適切にIAM Policyをアタッチすることで認可する感じ(?)

[Docs - IAM の仕組みについて]( https://docs.aws.amazon.com/ja_jp/IAM/latest/UserGuide/intro-structure.html ) を読むと良い。
[DevelopersIO - よくわかる認証と認可]( https://dev.classmethod.jp/articles/authentication-and-authorization/ ) も勉強になる。

### 用語

* https://docs.aws.amazon.com/IAM/latest/UserGuide/intro-structure.html 
* https://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles_terms-and-concepts.html
に基本的な用語や概念の説明がある。

#### IAMリソースのくくり

[AWS再入門ブログリレー2022 AWS IAM編]( https://dev.classmethod.jp/articles/re-introduction-2022-aws-iam/#toc-8 ) が参考になる。

* IAM Resources
  * > The user, group, role, policy, and identity provider objects that are stored in IAM. As with other AWS services, you can add, edit, and remove resources from IAM.
* IAM Identities
  * > The IAM resource objects that are used to identify and group. You can attach a policy to an IAM identity. These include users, groups, and roles.
    * IAM user / IAM group / IAM role
* IAM Entities
  * > The IAM resource objects that AWS uses for authentication. These include IAM users and roles.
  * IAMリソースのうち**認証**に使われるもの
  * IAM user と IAM role
* Principals
  * > A person or application that uses the AWS account root user, an IAM user, or an IAM role to sign in and make requests to AWS. Principals include federated users and assumed roles.
  * > A principal is a person or application that can make a request for an action or operation on an AWS resource. **The principal is authenticated as the AWS account root user or an IAM entity to make requests to AWS.** 
    > As a best practice, do not use your root user credentials for your daily work. Instead, create IAM entities (users and roles). You can also support federated users or programmatic access to allow an application to access your AWS account.
  * IAMユーザーやIAMロールといったIAMリソースを使ってAWSの各種サービスを利用する人やアプリケーションのこと。

#### その他
* Federated User
  * 「フェデレーション」はサービス間のユーザー認証連携のこと
    * 「外部 ID プロバイダー（Amazon/Facebook/Google/Github/etc）と AWS との間に信頼関係を作成すること」と言ってもいい
    * OIDC (OpenID Connect)およびSAML 2.0 (Security Assertion Markup Language)互換のIdP (Identity Provider) であれば良い
  * [Docs - ID プロバイダーとフェデレーション]( https://docs.aws.amazon.com/ja_jp/IAM/latest/UserGuide/id_roles_providers.html )
  * 関連用語
    * OpenID Connect (OIDC)
    * SAML2.0 (Security Assertion Markup Language)
  * > A federation is a group of computing or network providers agreeing upon standards of operation in a collective fashion.
https://en.wikipedia.org/wiki/Federation_(information_technology)
A federated identity in information technology is the means of linking a person’s electronic identity and attributes, stored across multiple distinct identity management systems.
https://en.wikipedia.org/wiki/Federated_identity
* Request
  * Principalがマネジメントコンソール・API・CLIを利用しようとすると、以下の情報からなるRequestをAWSへ送ることになる
    * Actions or operations - Principalが実行したいマネジメントコンソール上のActionやCLI/APIのOperation
    * Resources - Action/Operationの対象となるAWSリソースオブジェクト
    * Principal - Requestを送った人やアプリケーションが利用しているエンティティの情報。プリンシパルが利用しているエンティティに紐づいたポリシーの情報も含まれる
    * Environment data - IPアドレス、ユーザーエージェント、SSLステータス、日時
    * Resource data - リクエストされたリソースに関するデータ
  * 上記の情報が *request context* としてAWSに送られ、認証や認可に利用される
* Authentication
  * AWSにリクエストを送るためには、プリンシパルは認証情報を用いて認証されなければならない
  * 認証の仕方には以下の通りいくつかある
    * root userとしてコンソールから認証する：メールアドレスとパスワード
    * IAM Userとしてコンソールから認証する：Account ID(Alias)とusernameとパスワード
    * APIやCLIから認証する：access keyとsecret key
  * MFA(Multi Factor Authentication, 多要素認証)を利用することで認証におけるセキュリティーが増す
* Authorization
* Assume Role
* Pass Role


## ID - Identity


### type of identity
https://docs.aws.amazon.com/ja_jp/IAM/latest/UserGuide/id.html

認証に使われるIAMリソース。以下がある。

* AWS account root user
* IAM users
* IAM user groups
* IAM roles
* Temporary credentials in IAM
  * STSで発行されるやつ
  * 詳細はSTSの節参照

### AWS account root user

> Amazon Web Services (AWS) アカウントを初めてを作成する場合は、このアカウントのすべての AWS サービスとリソースに対して完全なアクセス許可を持つ 1 つの ID で始めます。このアイデンティティは、AWS アカウントのルートユーザーと呼ばれます。アカウントの作成に使用した E メールアドレスとパスワードを使用して、ルートユーザーとしてサインインできます。
> https://docs.aws.amazon.com/ja_jp/IAM/latest/UserGuide/id_root-user.html


* > 日常的なタスクには、それが管理者タスクであっても、**ルートユーザーを使用しないことを強くお勧めします**。代わりに、初期の IAM ユーザーを作成するためにのみ、ルートユーザーを使用するというベストプラクティスに従います。
* arnは `arn:aws:iam::123456789012:root` の形式

### IAM users

> An AWS Identity and Access Management (IAM) *user* is an entity that you create in AWS to represent the person or application that uses it to interact with AWS. A user in AWS consists of a name and credentials.
> ...
> An IAM user is a resource in IAM that has associated credentials and permissions. An IAM user can represent a person or an application that uses its credentials to make AWS requests. This is typically referred to as a **service account**. 
> If you choose to use the long-term credentials of an IAM user in your application, **do not embed access keys directly into your application code.** The AWS SDKs and the AWS Command Line Interface allow you to put access keys in known locations so that you do not have to keep them in code.
> https://docs.aws.amazon.com/IAM/latest/UserGuide/id_users.html

AWSを利用するユーザーやアプリケーションを表すエンティティで、名前と認証情報（コンソールパスワード、アクセスキー）で構成される。

IAM userを利用してAWSへアクセスするための認証方法はいくつかある。

* コンソールパスワード
  * マネジメントコンソールなどのインタラクティブセッションにサインインするときに使うもの
  * ID / passsword
  * マネジメントコンソールに入る時に使うやつ
* access key id / secret access key id
  * プログラム・コマンドラインが使う奴

認証を強固にするために、MFA（他要素認証）の設定をしたりもできる。


### IAM user groups


### IAM roles

AWSアカウントで作成できる特定のアクセス権限を持ったIAM Identityのこと。
認証の基礎となるAWSリソースという意味ではIAM userと似ているが、以下のいくつかの点で異なる。

* IAM userは基本的に1人の特定の人が利用することを想定しているが、IAM roleは必要とする任意の人が利用できるように設定できる
* IAM userは長期的な認証情報（パスワードやアクセスキー）で認証を行うが、IAM roleは一時的な認証情報で認証を行う

その他ポイント

* "assume role" は「roleを引き受ける」みたいなニュアンス
  * 「IAM roleで作業する」と考えている実体は「IAM roleを引き受けたセッション」
* [Docs - Roles terms and concepts]( https://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles_terms-and-concepts.html )
* [IAMロール徹底理解 〜 AssumeRoleの正体]( https://dev.classmethod.jp/articles/iam-role-and-assumerole/ )
* [ロールの用語と概念]( https://docs.aws.amazon.com/ja_jp/IAM/latest/UserGuide/id_roles_terms-and-concepts.html )
  * 信頼ポリシー / Trust policy
    * 誰/何がそのroleを引き受ける(assume)することができるかを定義する
    * Terraformでいう[aws_iam_role.assume_role_policy]( https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/iam_role#assume_role_policy )
    * Consoleでは"Trust relationships"タブで確認できるやつ
  * 許可ポリシー / Permission policy
    * Identity-based policiesとしてattachするやつ
  * プリンシパル / Principal
    * AWSで何かしらのactionを実行したりリソースへアクセスできるentityのこと
    * ex. AWS account root user /  IAM user / IAM role


#### Assume Role
stsの節参照


#### Pass Role

多くのAWSサービスを利用するためには、そのサービスにIAM roleを *渡す* 必要がある。例えば、EC2インスタンス・Lambda関数・ECS、Batchなどなど、それらのサービスが使うIAM roleを設定しておかないといけない。

* PassRoleを設定する際のPolicyの雛形
  * 「どのIAM role」を「どのサービス/どのリソース」に渡せるか、を記述する
      ```json
      {
          "Effect": "Allow",
          "Action": "iam:PassRole",
          "Resource": "arn:aws:iam::account-id:role/EC2-roles-for-XYZ"
          "Condition": {
              "StringEquals": {"iam:PassedToService": "ec2.amazonaws.com"},
              "StringLike": {
                  "iam:AssociatedResourceARN": [
                      "arn:aws:ec2:us-east-1:111122223333:instance/*",
                      "arn:aws:ec2:us-west-1:111122223333:instance/*"
                  ]
              }
          }
      }
      ```
* 例えば `AmazonEC2FullAccess` には `iam:PassRole` は含まれていないのはなぜか？
  * EC2インスタンスを自由に立ち上げられて、かつPassRoleでインスタンスに自由な権限をつけられる状態であると、インスタンスのに強いRoleを設定しそれを使うことで権限昇格することができうる
  * こういった観点から、EC2などのサービス自体の諸々を利用する権限と、そのサービスにroleを渡す権限（iam:PassRole）は分かれているのだと思う
* 具体例
  * IAM user `user_a` がEC2インスタンスを利用するために必要な設定
    * まずEC2インスタンスが使うことになるIAM role `EC2-roles-for-XYZ` を用意する
      * attachするIAM policy
        ```json
        {
            "Version": "2012-10-17",
            "Statement": {
                "Effect": "Allow",
                "Action": [ "A list of the permissions the role is allowed to use" ],
                "Resource": [ "A list of the resources the role is allowed to access" ]
            }
        }      
        ```
      * 設定すべきtrust relationshipは以下
        ```json
        {
            "Version": "2012-10-17",
            "Statement": {
                "Sid": "TrustPolicyStatementThatAllowsEC2ServiceToAssumeTheAttachedRole",
                "Effect": "Allow",
                "Principal": { "Service": "ec2.amazonaws.com" },
                "Action": "sts:AssumeRole"
            }
        }                    
        ```
    * IAM user `user_a` に PassRole を許可する以下のpolicyをattach
        ```json
        {
            "Version": "2012-10-17",
            "Statement": [{
                "Effect": "Allow",
                "Action": [
                    "iam:GetRole",
                    "iam:PassRole"
                ],
                "Resource": "arn:aws:iam::account-id:role/EC2-roles-for-XYZ"
            }]
        }
        ```
    * これで `user_a` は `EC2-roles-for-XYZ` を紐づけてEC2インスタンスを利用できる
* references
  * [Docs - IAM ロールを特定の AWS のサービスに渡す]( https://docs.aws.amazon.com/ja_jp/IAM/latest/UserGuide/reference_policies_examples_iam-passrole-service.html )
  * [Docs - AWS のサービスにロールを渡すアクセス権限をユーザーに付与する]( https://docs.aws.amazon.com/ja_jp/IAM/latest/UserGuide/id_roles_use_passrole.html )


### instance profile

> ロールを使用して、EC2 インスタンスで実行されるアプリケーションにアクセス権限を付与するには、少しの追加設定が必要です。EC2 インスタンスで実行されるアプリケーションは、仮想化されたオペレーティングシステムによって AWS から抽象化されます。この追加の分離のため、**AWS ロールとその関連付けられたアクセス許可を EC2 インスタンスに割り当て、アプリケーションに対してそれらの使用を許可する別手順が必要になります。この別手順は、インスタンスにアタッチされるインスタンスプロファイルの作成です。** インスタンスプロファイルは、ロールを含んでおり、インスタンスで実行されるアプリケーションにロールの一時的な認証情報を提供できます。それらの一時的な認証情報は、アプリケーションの API コールで、リソースへのアクセスを許可するために、または、ロールで指定されたリソースのみにアクセスを制限するために使用できます。同時に EC2 インスタンスに割り当てることができるのは 1 つのロールだけです。インスタンスのすべてのアプリケーションは、同じロールとアクセス権限を共有します。
> https://docs.aws.amazon.com/ja_jp/IAM/latest/UserGuide/id_roles_use_switch-role-ec2.html

terraformでは [aws_iam_instance_profile]( https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/iam_instance_profile ) で作成できる。

[Qiita - インスタンスプロファイルを理解するついでにポリシーとロールを整理する。]( https://qiita.com/sakai00kou/items/a4b96dcfa6bb3e656cd9 )

### IAM user vs IAM role

IAM userとIAM roleの対比は認証情報が長期的か一時的かの対比に近い。stsの節も参照。

* [IAM ユーザーの作成が適している場合 (ロールではなく)]( https://docs.aws.amazon.com/ja_jp/IAM/latest/UserGuide/id.html#id_which-to-choose )
* [IAM ロールの作成が適している場合 (ユーザーではなく)]( https://docs.aws.amazon.com/ja_jp/IAM/latest/UserGuide/id.html#id_which-to-choose_role )

### 混乱した代理問題と外部ID

英語では "The confused deputy problem"

* Docs
  * https://docs.aws.amazon.com/ja_jp/IAM/latest/UserGuide/confused-deputy.html
  * https://docs.aws.amazon.com/ja_jp/AmazonRDS/latest/UserGuide/cross-service-confused-deputy-prevention.html
  * https://docs.aws.amazon.com/ja_jp/IAM/latest/UserGuide/id_roles_create_for-user_externalid.html
* Blogs
  * https://dev.classmethod.jp/articles/iam-role-externalid/
  * https://qiita.com/hkak03key/items/a960b7523557f03bc098



ポイント
* 「混乱した代理問題」というセキュリティで有名な問題がある
  * AWSベースのサードパーティーのサービスを利用する際に、そのサードパーティーのAWSアカウント全体をPrincipalにして自分のAWSアカウントのIAM Roleの設定をしてしまうと、サードパーティーの誰でも自分のAWSアカウントのIAM Roleが使えてしまうよね、みたいな問題
  * サードパーティーサービスの利用者の識別が必要、的な話
  * 外部IDはこの問題の対策として重要
* サードパーティーの例としてはSnowflakeなど
* AWSは外部IDを機密情報としては扱わない
  * 外部IDはサードパーティー利用者の識別のため使われるもので、**サードパーティーサービス自身が**利用者ごとに一意となるように管理しているということが大事
  * サードパーティーサービスのユーザー名みたいなものなのでこれ自体は機密情報ではないという整理になる


## STS - Security Token Service

### stsの基礎

https://docs.aws.amazon.com/IAM/latest/UserGuide/id_credentials_temp.html

STSは Temporary security credentials（一時的な認証情報、一時的なクレデンシャル）を作成・提供してくれるIAMの補助的なサービス。

一時的な認証情報の機能はIAM userが使用できる長期的な認証情報（アクセスキー）とほぼ同じだが、以下のような違いがある

* 一時的な認証情報は名前の通り、lifetimeが数分から数時間と短くなっている。有効期限が切れると、AWSはそれらを認証しなくなる
* 一時的な認証情報はユーザーと共に保存されることはなく、ユーザーのリクエストに応じて動的に生成され、提供される

これらの違いから、一時的な認証情報を利用する利点は以下の通り。

* アプリケーションに認証情報を埋め込む必要がなく、また有効期限が限られているため不要になった際にローテーションしたり削除する必要がない
  * 例えば退職者が出たのにともない、認証情報をローテーション・削除する、といったことが必要なくなる
* 利用者に対して、IAM Identityを用意せずAWSリソースへのアクセスさせることが可能となる
  * IAM roleやID Federationはこの一時的な認証情報を基に機能している

以下のブログは非常に勉強になる

* https://dev.classmethod.jp/articles/re-introduction-2022-aws-iam/
* https://dev.classmethod.jp/articles/getfederetiontoken-assumerole-getsessiontoken/


### stsのaction

https://docs.aws.amazon.com/ja_jp/IAM/latest/UserGuide/id_credentials_temp_request.html

#### AssumeRole

#### AssumeRoleWithWebIdentity

#### GetSessionToken 

MFAが有効なIAM userに対し、MFAデバイスで正常に認証が済んでいることになった一時的な認証情報を返してくれる。
この一時的な認証情報を適切に使えばMFAデバイスによる認証の回数を減らすことができる。


## Policy
IAM ID (User/Group/Role) に対する認可の管理をするもの。
「Action（どのサービスの）」「Resource（どういう機能や範囲を）」「Effect（許可/拒否）」という3つの観点で様々な権限を表現する。

Policyに関しては [ドキュメント - IAM でのポリシーとアクセス許可]( https://docs.aws.amazon.com/ja_jp/IAM/latest/UserGuide/access_policies.html ) をまず読むと良い。

### type of policy

IAM Policyを使用頻度順に分類すると以下の通り。

* **Identity-based policies**
  * IAM user / group / role にアタッチするpolicy
  * 上記各IAM Identityが実行できる内容を制御・制限するのに使う
  * **AWS Managed Policy / Customer Managed Policy / Inline Policy の3種類ある**
* **Resource-based policies**
  * アタッチされているリソースに対して特定のアクションを実行するために指定されたプリンシパルのアクセス許可を付与するとともに、このアクセス許可が適用される条件を定義するのに使う
  * **Inline policy**をリソースにアタッチする
  * 次の具体例を見てもわかるように、cross-account-accessを許可するために使うことが多い
  * メジャーなのは次の二つ
    * S3のバケットポリシー
      * terraformで言うと[aws_s3_bucket_policy]( https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/s3_bucket_policy )などで設定できる
    * IAM roleの信頼ポリシー (Trust policy, assume_role_policy)
      * terraformで言うと[aws_iam_role の assume_role_policy(required)]( https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/iam_role#assume_role_policy )
* 【まだ使ったことなし】Permissions boundaries
* Organizations SCPs
  * Service Control Policy の略（[Documents]( https://docs.aws.amazon.com/ja_jp/organizations/latest/userguide/orgs_manage_policies_scps.html )）
  * 全社に渡るDenyのルールなどが書いてある感じ
* 【まだ使ったことなし】ACL: Access Control Lists
* 【まだ使ったことなし】Session Policies

https://docs.aws.amazon.com/ja_jp/IAM/latest/UserGuide/access_policies_identity-vs-resource.html

### example

* [IAM アイデンティティベースのポリシーの例]( https://docs.aws.amazon.com/ja_jp/IAM/latest/UserGuide/access_policies_examples.html )
* [Amazon S3: 特定の S3 バケットの管理を制限する]( https://docs.aws.amazon.com/ja_jp/IAM/latest/UserGuide/reference_policies_examples_s3_deny-except-bucket.html )
    ```json
    {
        "Version": "2012-10-17",
        "Statement": [
            {
                "Effect": "Allow",
                "Action": "s3:*",
                "Resource": [
                    "arn:aws:s3:::bucket-name",
                    "arn:aws:s3:::bucket-name/*"
                ]
            },
            {
                "Effect": "Deny",
                "NotAction": "s3:*",
                "NotResource": [
                    "arn:aws:s3:::bucket-name",
                    "arn:aws:s3:::bucket-name/*"
                ]
            }
        ]
    }
    ```

### Rules of JSON policy document 
詳細は [IAM JSON ポリシーの要素のリファレンス]( https://docs.aws.amazon.com/ja_jp/IAM/latest/UserGuide/reference_policies_elements.html ) を参照。

トップレベルで指定できるのは

* version
  * `2012-10-17` か `2008-10-17` が指定できるが基本的に前者を明示して使う
* statement
  * policyの本体
  * 複数のステートメントを `"Statement": [{...},{...},{...}]` の形で書ける

statementの中身には

* Sid (Statement ID)
* Effect
  * Allow or Deny
* Action
  * `"Action": "s3:GetObject"` などのように許可(拒否)するAWSサービスのオペレーション（API）を記載する
  * どのようなものがあるかは各種API Referenceを見れば良い
    * https://docs.aws.amazon.com/ja_jp/service-authorization/latest/reference/reference_policies_actions-resources-contextkeys.html
    * 例： [ドキュメント - S3 API reference]( https://docs.aws.amazon.com/ja_jp/AmazonS3/latest/API/API_Operations_Amazon_Simple_Storage_Service.html )
* Resource
  * Actionの対象となるリソースをARNで指定する
  * ARNの詳細は [IAM ARN]( https://docs.aws.amazon.com/ja_jp/IAM/latest/UserGuide/reference_identifiers.html#identifiers-arns ) で確認できる
* Condition
  * ポリシーが実行されるのに必要な条件の記述
  * 条件の具体例
    * IPアドレスの制限
    * Github actionsから来るrepo情報を用いた制限
  * 「[複数のキーまたは値による条件の作成]( https://docs.aws.amazon.com/ja_jp/IAM/latest/UserGuide/reference_policies_multi-value-conditions.html )」にあるように、 `ForAllValues` や `ForAnyValue` を使うことで、条件の対象を複数にできる
* Principal
  * **リソースベースポリシーで必要になるやつ**
  * Principal自体はAWSで何かしらのactionを実行したりリソースへアクセスできるIAM entityのこと
  * IAMロールの信頼ポリシーでは、Principal要素に誰がこのロールを引き受けることができるかを指定できる
  * リソースベースのポリシーでは、Principal要素に誰がそのリソースへアクセスできるかを指定できる



### ID based policy

IDベースのポリシーはIdentity (user/group/role) が実行できるアクション、リソース、および条件を制御するために使うjsonで書かれた文書。
ID based policyには3種類ある。
https://docs.aws.amazon.com/ja_jp/IAM/latest/UserGuide/access_policies_managed-vs-inline.html

* AWS Managed policy
* Customer Managed policy
* Inline policy

#### AWS Managed Policy

* AWSが作成・管理している *Standalone policy*
  * *Standalone policy* とは、AWSリソースとしてarn ( ex.: `arn:aws:iam::aws:policy/IAMReadOnlyAccess` ) を持っているpolicyということ
  * IAMのページの Policiesから検索でき、Policy名の横にロゴがついてる
* AWS Managed Policy の中身は編集できない
* ex.) AmazonS3FullAccess

#### Customer Managed Policy

https://docs.aws.amazon.com/ja_jp/IAM/latest/UserGuide/access_policies_managed-vs-inline.html#customer-managed-policies

* アカウントごとに作成・管理できる *Standalone policy*
  * AWS Managed Policyと同様、arnを持ち、IAMのページのPoliciesで検索可能
* terraformについてのメモ
  * [Resource: `aws_iam_policy`]( https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/iam_policy ) を使うことでManaged Policyは作成できる
  * [Resource: `aws_iam_role_policy_attachment`]( https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/iam_role_policy_attachment ) でManaged Policy を IAM role に attach できる
  * [Resource: `aws_iam_policy_attachment`]( https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/iam_policy_attachment ) でManaged Policy を IAM role/user/group に attach できる


#### Inline Policy

https://docs.aws.amazon.com/ja_jp/IAM/latest/UserGuide/access_policies_managed-vs-inline.html#inline-policies

* IAM Identity (user / group / role) に埋め込まれたポリシー
* policyとしてarnを持っていたりするわけではない
* 実際に (Customer) Managed policy は Policy のページへのリンクがついているが、 inline policy は この IAM role に埋め込まれているのでそういったリンクはない
* terraformについてのメモ
  * [Resource: `aws_iam_role_policy`]( https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/iam_role_policy ) を使うことで作成できる
    * `aws_iam_role_policy` 内で埋め込む role は直接指定する


#### Managed vs. Inline
https://docs.aws.amazon.com/ja_jp/IAM/latest/UserGuide/access_policies_managed-vs-inline.html#choosing-managed-or-inline


### resource based policy

* [DevelopersIO - 特定の IAM ロールのみアクセスできる S3 バケットを実装する際に検討したあれこれ]( https://dev.classmethod.jp/articles/s3-bucket-acces-to-a-specific-role/ )
* [DevelopersIO - IAMロールセッション名にユーザー名を強制できる条件 sts:RoleSessionName が使えるようになりました]( https://dev.classmethod.jp/articles/sts-sessionname/ )

### policy evaluation

https://docs.aws.amazon.com/ja_jp/IAM/latest/UserGuide/reference_policies_evaluation-logic.html

特にクロスアカウントアクセス（1 つのアカウントのプリンシパルが別のアカウントのリソースにアクセスすること）する際に、Resource based policyが重要になってくる。
[Document - クロスアカウントポリシーの評価論理]( https://docs.aws.amazon.com/ja_jp/IAM/latest/UserGuide/reference_policies_evaluation-logic-cross-account.html )

アカウントAのIAM user `iam_principal_a` が、アカウントBのリソース（IAM role / S3 Bucket/ etc）へアクセする場合、

* 利用するプリンシパル `iam_principal_a` には ID based policy でアカウントBへのアクセスを許可
* 利用されるリソースには resource based policy でアカウントAからのアクセスを許可

という形で、利用する側・される側両者の設定が必要。
このようにResouce based policyは利用されるリソースの権限を明記する。



#### within a single account
https://docs.aws.amazon.com/ja_jp/IAM/latest/UserGuide/reference_policies_evaluation-logic.html#policy-eval-basics

#### cross account
https://docs.aws.amazon.com/ja_jp/IAM/latest/UserGuide/reference_policies_evaluation-logic-cross-account.html


### policy in terraform

#### json policy documentの記載方法
policyの記述の仕方はいくつかある

* jsonファイルとして外部に記述し、`policy = file("./ddb-allow-policy.json")`的な形で読み込む
    * templateとして使ったりするなら良さそう(?)
* `policy = jsonencode({...})` とjsonencodeで直接書いたjsonを囲ってあげる
    * ハイライトされたりして綺麗だし、コメントも書いたりできるのでこれがおすすめ
    * [Terraformのjsonencode関数にはJSONを入れても動くよ]( https://qiita.com/kanga/items/1ae96b7da2a7d76b070e )
* ヒアドキュメント（`  policy = <<EOT...` 的なやつ）で tf のファイルに直接jsonを書く
    * https://www.terraform.io/docs/language/expressions/strings.html#heredoc-strings
* `iam_policy_document` という data source を用意して参照する
    * planの差分が見にくいらしい

[DevelopersIO - TerraformでIAM Policyを書く方法5つ]( https://dev.classmethod.jp/articles/writing-iam-policy-with-terraform/ )


#### roleにpolicyをattachする方法

IAM roleにmanaged policyをattachする方法はいくつかある

1. `aws_iam_role` の属性 managed_policy_arns に直接arnを書く
2. `aws_iam_policy_attachment` リソースを使う
3. `aws_iam_role_policy_attachment` リソースを使う

それぞれ注意が必要で、

* `aws_iam_policy_attachment` を使う場合には、あるpolicyのattachは全てここに記述しないといけない
* `aws_iam_policy_attachment` と `aws_iam_role_policy_attachment` を同時に使うと差分が出続ける
* `aws_iam_role` のmanaged_policy_arns と `aws_iam_role_policy_attachment` を同時に使うと差分が出続ける
* `aws_iam_role` のmanaged_policy_arns と `aws_iam_policy_attachment` を同時に使うと差分が出続ける


### Organizations SCPs

* Service Control Policy の略（[Documents]( https://docs.aws.amazon.com/ja_jp/organizations/latest/userguide/orgs_manage_policies_scps.html )）
* 全社に渡るDenyのルールなどを規定するのに使える感じ


## Access Analyzer

* https://docs.aws.amazon.com/IAM/latest/UserGuide/what-is-access-analyzer.html
* [AWS CloudTrail の履歴から最小限の IAM Policy が作れる機能ができたようなので試してみた]( https://zenn.dev/mryhryki/articles/2021-04-08-aws-iam-policy )


## AWS SSO
* https://dev.classmethod.jp/articles/aws-sso-wakewakame/



## Tips
### aws-cli のデバッグ

以下のコマンドを使いながら、使っているIAMの整理をしつつデバッグすると良い

#### 現在使っているIAM情報を確認する
`get-caller-identity` はどんな権限もいらないし、明示的なdenyが入っていても使える珍しいやつ。以下のようにUserIdやAccout、ARNの情報が確認できるのでデバッグに使える。
```shell
$ aws sts get-caller-identity
{
    "UserId": "AIDASAMPLEUSERID",
    "Account": "123456789012",
    "Arn": "arn:aws:iam::123456789012:user/DevAdmin"
}
```
https://docs.aws.amazon.com/cli/latest/reference/sts/get-caller-identity.html

#### configの一覧
使われているconfigを以下の通り一覧表示できる。
```shell
$ aws configure list
      Name                    Value             Type    Location
      ----                    -----             ----    --------
   profile                <not set>             None    None
access_key     ****************3YE6      assume-role
secret_key     ****************a7UD      assume-role
    region           ap-northeast-1      config-file    ~/.aws/config
```
https://docs.aws.amazon.com/ja_jp/cli/latest/userguide/cli-configure-files.html

#### 設定と優先順位
AWSの認証情報はいくつか設定の仕方があり、優先順位は以下のページの通り決まっている。
https://docs.aws.amazon.com/ja_jp/cli/latest/userguide/cli-configure-quickstart.html#cli-configure-quickstart-precedence

1. command line option
2. 環境変数
   * 詳細は[こちら]( https://docs.aws.amazon.com/ja_jp/cli/latest/userguide/cli-configure-envvars.html )、代表的な環境変数は以下
     * `AWS_PROFILE` : 使用するプロファイル名を指定
     * `AWS_ACCESS_KEY_ID` / `AWS_SECRET_ACCESS_KEY`
     * `AWS_CONFIG_FILE` : cliが使うconfigのファイルで、デフォルトは `~/.aws/config`
3. CLI credentials
    * Macのデフォルトは `~/.aws/credentials`
4. CLI configs
    * Macのデフォルトは `~/.aws/config`
5. Container credentials
    * Amazon ECS の task def に紐づけられるIAM role
6. Amazon EC2 instance profile credentials


#### 名前付きプロファイル
https://docs.aws.amazon.com/ja_jp/cli/latest/userguide/cli-configure-profiles.html

#### configファイルとcredentialsファイル

https://docs.aws.amazon.com/cli/latest/userguide/cli-configure-files.html

* `aws configure` コマンドはセンシティブな情報（ `aws_access_key_id` / `aws_secret_access_key` / `aws_session_token` ）は `credentials` ファイルに、そうではないその他の情報は `config` ファイルに配置する
* `config` ファイルでIAM roleが指定されている場合にはAWS STS AssumeRoleオペレーションを呼び出し、一時的な認証情報を得て  `~/.aws/cli/cache` に保存する。これによってIAM Roleの権限が使えるようになる。有効期限が切れるとキャッシュは削除され、サイドCLIによって自動的に認証情報が更新される。
* configファイルで各profileに記載することができるconfig変数
  * `role_arn` : assume role したい IAM role の arn
  * `mfa_serial` : assume role する際に利用するMFAデバイスのシリアル番号
  * `role_session_name` : セッション名を指定できる
    * [DevelopersIO - AWS CLIがAssumeRoleする際のセッション名を指定する]( https://dev.classmethod.jp/articles/aws-cli-assume-role-with-session-name/ )


#### その他

* CLIを使っている場合には [Docs - AWS CLI に関連するエラーのトラブルシューティング]( https://docs.aws.amazon.com/ja_jp/cli/latest/userguide/cli-chap-troubleshooting.html ) も参考になりそう

### Github actionsでOIDCを利用しAWSへアクセスする
https://docs.github.com/ja/actions/deployment/security-hardening-your-deployments/configuring-openid-connect-in-amazon-web-services

OpenID Connect(OIDC) を利用することで、Github ActionsでAWSのリソースへアクセスする際に、credential情報を保存する必要がなくなる



### Understanding unique ID prefixes
> An AWS-Access-Key-ID always begins with `AKIA` for IAM users or `ASIA` for temporary credentials from Security Token Service, as noted in IAM Identifiers in the AWS Identity and Access Management User Guide.
> https://stackoverflow.com/questions/58247672/how-to-fix-authorizationheadermalformed-when-calling-the-getobject-operation-e

> Understanding unique ID prefixes
> IAM uses the following prefixes to indicate what type of resource each unique ID applies to.
> https://docs.aws.amazon.com/IAM/latest/UserGuide/reference_identifiers.html#identifiers-unique-ids

IAMでは各リソースのIDのprefixがリソースごとに決まっている。

| Prefix | Resource Type |
| :---: | :---: |
| AIDA | IAM user |
| AROA | Role |
| AKIA | Access Key |
| ASIA | Temporary access key IDs |

AccessKeyIDや、get-caller-identityをした際のUserIdなどはここのルールに従っているはず。

### IAM Userの削除

適当にIAM Userを削除しようとすると

> Cannot delete entity, must remove tokens from principal first.

と出ることがある。IAM Userの削除前にUserに紐づく以下を事前に削除しておかないといけないという話。

* MFAトークン
* マネジメントコンソール
* Access Key


### IAMのarn一覧

各リソースのarnは以下の形式。
```
arn:aws:iam::account:root  
arn:aws:iam::account:user/user-name-with-path
arn:aws:iam::account:group/group-name-with-path
arn:aws:iam::account:role/role-name-with-path
arn:aws:iam::account:policy/policy-name-with-path
arn:aws:iam::account:instance-profile/instance-profile-name-with-path
arn:aws:sts::account:federated-user/user-name
arn:aws:sts::account:assumed-role/role-name/role-session-name
arn:aws:iam::account:mfa/virtual-device-name-with-path
arn:aws:iam::account:u2f/u2f-token-id
arn:aws:iam::account:server-certificate/certificate-name-with-path
arn:aws:iam::account:saml-provider/provider-name
arn:aws:iam::account:oidc-provider/provider-name
```
https://docs.aws.amazon.com/ja_jp/IAM/latest/UserGuide/reference_identifiers.html より


### boto3関係

#### デバッグログの出力

以下のようにdebugのログを出すことができる

```python
In [1]: import boto3, botocore
   ...: botocore.session.Session().set_debug_logger()  # これ
   ...: session = boto3.session.Session(profile_name='datahub-dev')
   ...: client = session.client('sts')
   ...: client.get_caller_identity()

...
2022-11-30 19:10:35,539 - botocore.utils - DEBUG - IMDS ENDPOINT: http://169.254.169.254/
2022-11-30 19:10:35,542 - botocore.credentials - DEBUG - Skipping environment variable credential check because profile name was explicitly set.
2022-11-30 19:10:35,542 - botocore.credentials - DEBUG - Looking for credentials via: assume-role
2022-11-30 19:10:35,542 - botocore.credentials - DEBUG - Looking for credentials via: assume-role-with-web-identity
2022-11-30 19:10:35,542 - botocore.credentials - DEBUG - Looking for credentials via: sso
2022-11-30 19:10:35,543 - botocore.credentials - DEBUG - Looking for credentials via: shared-credentials-file
2022-11-30 19:10:35,543 - botocore.credentials - INFO - Found credentials in shared credentials file: ~/.aws/credentials
...
```


#### credentialsとconfigの指定

https://boto3.amazonaws.com/v1/documentation/api/latest/guide/credentials.html
https://boto3.amazonaws.com/v1/documentation/api/latest/guide/configuration.html



## References
* User Policy
  * [S3アクセスに関するユーザーポリシーの例]( https://docs.aws.amazon.com/ja_jp/AmazonS3/latest/dev/example-policies-s3.html )
  * [Amazon EC2: Allows starting or stopping EC2 instances a user has tagged, programmatically and in the console]( https://docs.aws.amazon.com/IAM/latest/UserGuide/reference_policies_examples_ec2_tag-owner.html )
* AWS training and certification
  * [Deep Dive with Security: AWS Identity and Access Management (IAM)]( https://explore.skillbuilder.aws/learn/course/external/view/elearning/9054/deep-dive-with-security-aws-identity-and-access-management-iam-japanese )
* [terraformでAWSのIAMを定義する方法]( https://zwzw.hatenablog.com/entry/terraform-iam )
* [DevelopersIO - AWS再入門ブログリレー2022 AWS IAM編]( https://dev.classmethod.jp/articles/re-introduction-2022-aws-iam/ ) ：めっちゃ深ぼってあり良い記事
* [DevelopersIO - GetFederationToken は AssumeRole や GetSessionToken と何が違うのか]( https://dev.classmethod.jp/articles/getfederetiontoken-assumerole-getsessiontoken/ )：めっちゃ細かい話が書いてあって良い記事
* [DevelopersIO - よくわかる認証と認可]( https://dev.classmethod.jp/articles/authentication-and-authorization/ ) も勉強になる。
