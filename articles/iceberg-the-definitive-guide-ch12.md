---
title: "Apache Iceberg: The Definitive Guide 12章 Governance and Security"
emoji: "🧊"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Iceberg", "DataEngineering", "OTF", "Snowflake"]
published: true
publication_name: dataheroes
---


## はじめに

こんにちは！ナウキャストでデータエンジニアをしているけびんです。

SnowVillage で行っている [Apache Iceberg: The Definitive Guid]( https://www.oreilly.com/library/view/apache-iceberg-the/9781098148614/ ) 輪読会の12章の資料です。12章は "Governance and Security" というタイトルで Iceberg に関連するセキュリティやガバナンスに関するトピックが書かれていますが、その内容をまとめたり、調べて見つけた関連した内容も適宜記載したりしています。

:::message
この形式が私の感想・コメントです
:::



## 12章の introduction

Iceberg は OTF でありメタデータファイルとデータファイルとでデータセットを構成する方法を定義してはいますが、セキュリティに関する詳細は（一部暗号化に関するプロパティなどはありますが）基本的に組み込まれていません。そのため、さまざまなレイヤで独自に設定する必要があります。

特に３つのレイヤで考えるのが大事です

* ファイル・ストレージレベルでのセキュリティ
* Semantic layer でのセキュリティとガバナンス
* Catalog レベルでのセキュリティとガバナンス


### Security 一般論

📖 セキュリティについて簡単にまとめておきます。

* JIS Q 27000 で定義されているのは以下
    * ３要素
        * **機密性 / Confidentiality**
            * 許可されていない人が情報にアクセスできないようになっていること
            * データが秘匿されていること
            * 関連技術：認証・認可、暗号化、など
        * **完全性 / Integrity**
            * 情報が改竄されたり消去されたりせず、正確かつ完全に残っていること
            * 関連技術：ハッシュ、署名、メッセージ認証符号、など
        * **可用性 / Availability**
            * 許可された人が望んだ時に情報にアクセスできるようになっていること
            * 関連技術：冗長化、バックアップ
    * あとから追加された要素
        * **真正性 / Authenticity**
          * 「エンティティはそれが主張するとおりのものであるという特性」と定義されている
          * 関連技術：認証
        * **責任追跡性 / Accountability**
            * インシデント発生時、その影響範囲や経路をエビデンスとともに厳密に特定できるようになっていること
            * Traceability と言ってもいいかも
            * 関連技術：ログ
        * **否認防止 / Non-repudiation**
          * 「主張された事象または処置の発生、およびそれらをひきおきしたエンティティを証明する能力」と定義されている
        * **信頼性 / Reliability**
          * 「意図する行動と結果とが一貫しているという特性」と定義されている
* 多層防御 / Multilayer Defense
  * 近年攻撃の手口は多様化しており、不正アクセスに対する入り口の対策など単一的なセキュリティ対策だけでは不十分である。不正アクセス後の対策にも目を向けて、複数の層で対策を講じておくべきという考え方が一般的で、多層防御と呼ばれたりします。


:::message
多層防御の考え方に基づくと３つのレイヤのセキュリティを適切に組み合わせるのが大事だと考えています。一方で適当に組み合わせてしまうと管理が大変にもなりそうなので、適切なセキュリティ・ガバナンスの設計が大事になると思います。
:::


### Governance 一般論

📖 DMBOK2 ではデータガバナンスは以下のように定義されています。

> 職務権限を通してデータ資産管理を統制し、意思決定を共有させるため、データ管理に関する取り決め（原則、ポリシー、手続、評価指標、ツール、責任）を定義し、実施する



## Securing files

まずはファイルレベル・ストレージレベルでのセキュリティについてです。


### Securing Files: Best Practices

ファイルに関するセキュリティでのベストプラクティスは以下です。

* **Least privilege access / 最小権限の原則**
    * 人やシステムに対して、目前のタスクを実行するために必要十分な権限とデータだけを与えるということ
    * 機密性を担保し、また事故が起きた時のリスクを最小限にすることができる
    * 📖 定義的な最小性と時間的な最小性がある
        * 定義的な最小性
            * 人にもシステムにも必要な権限のみ付与する
        * 時間的な最小性
            * 特定の権限（特に強い権限）が永続的に必要なことはほとんどない
            * 作業するタイミングだけ権限を付与する
* **Encryption at rest and in transit / 暗号化**
    * 保存時と通信時両方でそれぞれ暗号化しよう
    * 機密性を担保し、万が一データが流出してもデータの中身を直接見られないようにしておく
* **Strong authentication and identity management**
    * IAM による認証・認可を適切に実行し機密性を担保すること
        * MFA設定しようとか強いパスワード設定しようとかも含む
* **Audit trails and logging / 監査証跡とログ**
    * 何か事故が起きた時に適切に検証できるようにし、責任追跡性を担保すること
    * 端的にいうとログはちゃんと記録しておこう、という話
* **Data retention and disposal policies**
    * 適切に保存期間や廃棄に関するポリシーを定めておくことで、保護対象を減らすことでリスクを減らすこと
    * コスト的にも不要なデータを削除することは大事
* **Continuous monitoring**
    * 継続的にモニタリングして、インシデントをリアルタイムで検出しましょう的な
    * いわゆる発見的統制の一種



ファイルレベルでのセキュリティに関する pros / cons は以下の通りです。

* pros
    * 柔軟なアクセス制御や暗号化の設定が可能で、認可されていないアクセスからのリスクを下げられる
* cons
    * ファイル数が増えるにつれ、管理対象が増え複雑になるためスケールしにくい
    * ユーザーやツールには簡潔なデータの見せ方が必要になるかも


:::message
ファイルレベルのセキュリティはこのあと見るように、各種ストレージの機能を利用するだけで実現可能なので比較的実現コストが低いかなと思います。一方でスケーラビリティの話もあるので他のセキュリティ設定との組み合わせが大事になりそうです。
:::


次からは具体的にストレージごとに設定項目を見ていきます。


### HDFS

まずは Hadoop Distributed File System のアクセス制御と暗号化について見ていきましょう。

#### Access Control 

アクセス制御は２つの方法が用意されています。

* Permission
    * HDFS は伝統的なファイルシステムと同様な権限管理の仕組みが採用されている
    * イメージとしては chmod や chown で linux でファイルの権限管理するのと同じような仕組み
    * ファイルやディレクトリにはその所有者であるユーザーとグループが決まっている
    * 所有ユーザー・所有グループ・それ以外のユーザーごとにアクセス権限を設定できる
        * ここが柔軟性が低いポイント
    * アクセス権限は read/write/execute の３種
* ACL: Access Control List
    * 通常のpermissionシステムだと、「所有ユーザー・所有グループ・それ以外のユーザーごとに」しかアクセス権限を設定できなかった
    * ACLを使うと、所有者以外のユーザーやグループに対しても権限が設定できるようになる
    * 具体例
        * 以下のような設定
            ```
               user::rw-
               user:bruce:rwx                  #effective:r--
               group::r-x                      #effective:r--
               group:sales:rwx                 #effective:r--
               mask::r--
               other::r--
            ```
        * 普通の permission 設定と同じところ
            * 所有ユーザー `user::rw-`
            * 所有グループ `group::r-x`
            * その他 `other::r--`
        * ACL で追加されているところ
            * ユーザー bruce `user:bruce:rwx`
            * グループ sales `group:sales:rwx`
            * マスク `mask::r--`
                * permission boundary 的な機能

詳細は以下などを参照してください。

https://manual.geeko.jp/ja/cha.security.acls.html

https://hadoop.apache.org/docs/r3.4.0/hadoop-project-dist/hadoop-hdfs/HdfsPermissionsGuide.html


#### Encryption

簡単に暗号化を利用できるように TDE と TZ という機能が用意されています。

* TDE: Transparent Data Encryption
    * end-to-end で投下的な暗号化がHDFSでは実装されている
    * client によってのみ暗号化・復号化され、HDFSには暗号化されていないデータは保存されず、通信も暗号化された状態で行われる
    * また、アプリケーションコードを変更することなくこの暗号化・復号化は透過的に（存在を意識しなくても）行われる
* EZ: Encryption Zone
    * 透過的な暗号化のために作られたのが Encryption Zone という仕組み
    * EZは特定のディレクトリに対して暗号化を必須にし、書き込み時に透過的に暗号化され、読み込み時には透過的に復号化されるようになっている

詳細は以下を参照してください。

https://hadoop.apache.org/docs/r3.4.0/hadoop-project-dist/hadoop-hdfs/TransparentEncryption.html


### Amazon S3

次に S3 についてです。

#### Encryption

Server-Side Encryption の方法が３つ提供されています。

* SSE-S3 (SSE with S3-managed keys)
    * 透過的によしなにやってくれるやつ
    * S3が管理する鍵で暗号化する
* SSE-KMS (SSE with the AWS KMS)
    * AWS KMS を利用して、暗号化に使うキーを指定して行う方法
    * SSE-S3 よりもより強固な要件が求められる時によく使う
* SSE-C (SSE with Customer-provided keys)
    * 顧客が提供するキーを利用して暗号化を行うやつ
    * AWSはキーを触れず一番安全ではあるが、鍵の管理などはすべて顧客がやらないといけない

この辺りの鍵の設定は Iceberg Table でも設定可能で、以下のようなプロパティが用意されています。
* s3.sse.type
* s3.sse.key
* s3.sse.md5


#### Access Control

アクセス制御の方法も複数用意しています。

* Bucket policies
    * いわゆる resource based policy 
    * バケットやオブジェクト側主体でアクセス制御を柔軟に設定できる
* IAM policy
    * いわゆる identity based policy
    * IAM user や role など主体でアクセス制御を柔軟に設定できる
    * アクセス可能なためには IAM policy でも Bucket policy でも許可されている必要がある
* Object ACLs
    * HDFS の ACL に近い
    * バケットやオブジェクトごとに、特定のアカウントからのアクセス権限を管理できる
    * https://dev.classmethod.jp/articles/amazon-s3-acl-basics/
    * アクセス主体の粒度も、権限の粒度も荒いため、基本的には IAM Policy / Bucket Policy を使うことが推奨されている

適切に AWS の IAM を理解して設定することが大事です。以下などを参照してください。

https://speakerdeck.com/kevinrobot34/introduction-aws-iam-3a810adc-5172-4460-9721-0041456eb2bf


#### その他

📖 他にも設定しておくと良いことはあるので紹介します。

* CloudTrail や Server Access Log
  * 責任追跡性を担保するには CloudTrail を利用したり、 S3 の Server Access Log の設定を行い、ログが出力され保存されるようにしておくことが重要です
  * また何かあった時に簡単にログを分析できるような環境を作っておくことも重要です。
* Block public access の設定
  * S3 はいろんな使い方ができるので public access が可能な状態にもできてしまう
  * データレイクとして利用する場合には block public access の設定をしておくのが大事


:::message
S3 のようなオブジェクトストレージは古くからあるサービスでさまざまな設定があるので適切に設定しておくことが大事だと思います。 Iceberg を利用すると、オブジェクトストレージのプラクティスを利用しやすいのは一つのメリットだなと感じました。
:::



### Azure Data Lake Storage

Azure Data Lake Storage でもほとんど同じです。暗号化とアクセス制御の設定は適切に行なっておきましょう。

* Encryption
  * Azure Key Vault を組み合わせることも可能
* Access Control
  * IAM(RBAC)
  * ACLs



### Google Cloud Storage

Google Cloud Storage でもほとんど同じです。暗号化とアクセス制御の設定は適切に行なっておきましょう。

* Encryption
  * Cloud KMS を組み合わせることも可能
* Access Control
  * IAM
  * Object ACLs



## Securing and Governing at the Semantic Layer


### Semantic Layer Best Practices


Semantic layer とはデータストレージと分析に使われるツール（BIツールなど）の間に位置する抽象化レイヤーのことで、エンドユーザーが一般的なビジネス用語を用いて自律的にデータにアクセスし分析できるようにするための、データのビジネス表現です。

![semantic-layer](/images/articles/iceberg-the-definitive-guide-ch12/semantic-layer.png =450x)
*https://www.dremio.com/blog/what-is-a-semantic-layer/ より*

さまざまな機能があり、セマンティックレイヤで以下のようなことを行うことが良いとされています。

* 📖ビジネスフレンドリーな語彙でデータを見ることができる
  * ビジネスユーザーが使い慣れた言葉に置き換え、データを活用しやすくする
* 📖物理的なデータ・データモデルの複雑さの隠蔽
  * 物理的なデータの詳細を知らなくてもデータを利用できるようにする
* 📖ビジネスロジックの中央化
  * 各所に散らばりがちなビジネスロジックを中央管理することで、計算方法が微妙に違うと言ったことを防ぐ
* アクセス制御の実現
  * 包括的なアクセス制御をRBACやABACで実現する
* データマスキング
  * PIIなどセンシティブなデータは適切にマスキングしておく
* データリネージュの管理
  * データの依存関係を追跡しておくことで、データの理解を助けるだけでなく、データの品質管理やコンプライアンスにも役に立つ
* あらゆるもののドキュメント化
  * ガバナンスやセキュリティのポリシー、アクセス制御、カタログ、リネージュなどあらゆるものに詳細なドキュメントをつけておくことで、透明性やコンプライアンス、トラブルシューティングに役に立つ

セマンティックレイヤーに関する詳細は以下などを参照してください。

https://zenn.dev/churadata/articles/e779a733c5fb35

https://www.dremio.com/blog/unified-semantic-layer-a-modern-solution-for-self-service-analytics/

https://www.dremio.com/blog/the-value-of-dremios-semantic-layer-and-the-apache-iceberg-lakehouse-to-the-snowflake-user/




semantic layer でのセキュリティの pros / cons には以下のようなものがあります。

* pros
    * 統一的なデータアクセス制御
    * 標準化したデータガバナンス・セキュリティポリシーの適用が可能
    * データを抽象化し、エンドユーザーがデータを簡単に利用できるようになる
* cons
    * セマンティックレイヤが単一障害点となる



### Dremio

Dremio の semantic layer は以下の機能を提供している

* Data lineage of virtual datasets
* Built-in wiki for documentation
* Role-, column-, and row-based access rules
    * より細かいアクセス制御を提供している
    * RBAC はよくあるやつ
    * column-based access control
        * PII を含むなど特定のカラムをマスキングすることができる
        * dremio だと udf を利用するようなイメージ
            ```sql
            -- Create the mask_salary UDF
            CREATE FUNCTION mask_salary(salary VARCHAR)
            RETURNS VARCHAR
            RETURN SELECT CASE 
                WHEN query_user()='user@example.com' OR is_member('HR') THEN salary
                ELSE 'XXX-XX'
            END;
            -- Create the employee_salaries table with the column masking policy
            CREATE TABLE employee_salaries (
                id INT,
                salary VARCHAR MASKING POLICY mask_salary (salary),
                department VARCHAR
            );
            ```
    * row-based access control
        * 特定のレコードは一部の人にしか見せたくないみたいな時に使える
        * これも dremio だと udf でやるイメージ
            ```sql
            -- Create the restrict_region UDF
            CREATE FUNCTION restrict_region(region VARCHAR)
            RETURNS BOOLEAN
            RETURN SELECT CASE 
                WHEN query_user()='user@example.com' OR is_member('HR') THEN true
                WHEN region = 'North' THEN true
                ELSE false
            END;
            -- Create the regional_employee_data table with the row access policy
            CREATE TABLE regional_employee_data (
                id INT,
                role VARCHAR,
                department VARCHAR,
                salary FLOAT,
                region VARCHAR,
                ROW ACCESS POLICY restrict_region(region)
            );
            ```



### Trino

Trino （以前は PrestoSQL という名だった）はオープンソースの分散SQLクエリエンジン。

Trino には federated data source の上に logic view というものを作ることができ、これを semantic layer として利用できる。これらの view のアクセス制御もできる。

view に関しては２つのセキュリティモードがある。
（Snowflake の stored procedure の owner / caller と同じ）

```
CREATE [ OR REPLACE ] VIEW view_name
[ COMMENT view_comment ]
[ SECURITY { DEFINER | INVOKER } ]
AS query
```

* DEFINER モード
    * view が参照しているテーブルはクエリ実行者ではなく、クエリの owner （viewの作成者もしくは定義者）の権限で実行される
* INVOKER モード
    * view が参照しているテーブルはクエリ実行者の権限を利用してアクセスされる

https://trino.io/docs/current/sql/create-view.html



RBAC も Snowflake と同様な感じでできる。


:::message
Semantic layer に詳しいわけではないですが、Dremioの例もTrinoの例も Semantic Layer 感はあんまりない気がします
:::



## Securing and Governing at the Catalog Level

カタログは Lakehouse アーキテクチャの重要な構成要素であり、メタデータを管理し、効率的なクエリ実行を可能にします。この節ではカタログをセキュリティ敵に保護し適切に管理する方法を確認します。

Iceberg におけるカタログについては5章で確認しました。詳細はこちらをご覧ください。

https://zenn.dev/musyu/articles/5d9ee475f5f51a

カタログレベルでセキュリティの設定をすると、どのクエリエンジンからアクセスされた際にも一貫したアクセス制御やセキュリティポリシーの適用が可能になり、データのクエリ・分析方法に依存しない統一したセキュリティを確保できるため非常に重要になります。


カタログレベでのセキュリティの pros / cons には以下のようなものがあります。
* pros
  * カタログはメタデータが中央に集められた場所なため、メタデータに基づいた一貫したセキュリティ・ガバナンスポリシーの適用が可能になる
  * テーブルやデータデータベースへの包括的なアクセス制御が実現可能
  * カタログはストレージを抽象化したレイヤなため、ファイルレベルよりもシンプルなアクセス制御が提供可能
* cons
  * カタログが単一障害点になる
  * 選択したカタログによって実現可能性が異なってしまう



### Nessie


### Tabular


### AWS Glue and Lake Formation





## Additional Security and Governance Considerations

セキュリティやガバナンスを考える際に file store level / semantic layer level / catalog level でそれぞれメリットデメリットがあることを見てきた。

以下のポイントを考慮しながらどういったセキュリティの設定をするかを考えていくのが重要である。

* Use case
    * ユースケースが違えばどのレベルでのセキュリティやガバナンスが適切かは変わる
    * 例えば機密性の高いデータであればカタログレベルが良いはずだが、そうでなければ semantic layer でやると良さそう、など
* Scalability
    * Scalability も大事なポイント
    * file level だと、データが増えるにつれて権限やガバナンスのポリシーを管理するのは大変になりやすい
* Performance
    * パフォーマンスについても大事
    * semantic layer はクエリの最適化につながるかもしれないが一方でオーバーヘッドを持ち込む可能性もある
* Data abstraction
    * どうやって抽象化・簡素化してユーザーやツールがアクセスしやすくするかを考えておくのも大事
* Operational overhead
    * オペレーションコストも無視できないので大事
* Redundancy and failover
    * 可用性と信頼性を担保するために冗長化などどうするかも考えないといけない



## まとめ


セキュリティやガバナンスを実現するための銀の弾丸はないです。
適切に各レイヤでのセキュリティ・ガバナンスを組み合わせ、組織の要件を満たせるようにするのが大事です。