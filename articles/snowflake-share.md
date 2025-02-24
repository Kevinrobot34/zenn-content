---
title: "Snowflake Share 徹底解説"
emoji: "🤝"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Snowflake", "DataEngineering", "SQL", "Share"]
published: true
publication_name: dataheroes
---

## はじめに

こんにちは！ナウキャストのデータエンジニアのけびんです。

Snowflake には様々な機能・特徴があり、以下のような絵で表現されることがよくあると思いますが、データやアプリの共有の部分は特に注目している方も多いのではないでしょうか？
![snowflake-arch](/images/articles/snowflake-share/snowflake-arch.png)
*https://www.snowflake.com/en/data-cloud/overview/cross-cloud-snowgrid/ より。SNOWGRIDとしてデータ共有に関する機能が外側に描かれている。*


ナウキャストでも Snowflake の Share を利用してお客様にデータを提供しているのですが、 Share 周辺は意外と難しく複雑なため今回の記事でまとめておこうと思います。


## Data Sharing に関する用語

まず最初に Snowflake でデータを共有する際によく出てくる単語を簡単にまとめておきます。

* Provider
  * 共有するデータの提供者
  * またはデータ提供者の Snowflake アカウント
* Consumer
  * 共有するデータの利用者
  * またはデータ利用者の Snowflake アカウント
* Share
  * Snowflake の名前付きオブジェクトで、本記事のメイントピック
    * 単にデータを共有するという意味なのか、名前付きオブジェクトのことなのかは文脈によるので注意
  * Database を共有するために必要な情報をカプセル化したもの
    * Share する Consumer の SNowflake Account やコメントなど
    * `SHOW SHARES` の結果を見るとイメージしやすいはず（[ドキュメント](https://docs.snowflake.com/en/sql-reference/sql/show-shares#examples)より）
        ```
        SHOW SHARES;

        +-------------------------------+----------+----------------------+---------------+-----------------------+------------------+--------------+----------------------------------------+---------------------+
        | created_on                    | kind     | owner_account        | name          | database_name         | to               | owner        | comment                                | listing_global_name |                  |
        |-------------------------------+----------+----------------------+---------------+-----------------------+------------------+--------------+----------------------------------------|---------------------|
        | 2016-07-09 19:18:09.821 -0700 | INBOUND  | SNOW.MY_TEST_ACCOUNT | SAMPLE_DATA   | SNOWFLAKE_SAMPLE_DATA |                  |              | Sample data sets provided by Snowflake |                     |
        | 2017-06-15 17:02:29.625 -0700 | OUTBOUND | SNOW.MY_TEST_ACCOUNT | SALES_S       | SALES_DB              | XY12345, YZ23456 | ACCOUNTADMIN |                                        |                     |
        +-------------------------------+----------+----------------------+---------------+-----------------------+------------------+--------------+----------------------------------------+---------------------+
        ```
  * Share が対象とする Database はあくまで一つで、共有するオブジェクトはその Database 内部に存在していないといけない、というのがポイントです。
* Direct Sharing
  * Share を利用して直接 Consumer にデータを共有する方法
  * 昔からある方法だが、クロスリージョン・クラウドなアカウント同士だと共有できないなどの制限もある



## Data Sharing の仕組み

まず最初に Snowflake での Data Sharing がどのような仕組みなのかを確認しておきましょう。

![data-sharing-overview](/images/articles/snowflake-share/data-sharing-overview.png)
*https://docs.snowflake.com/en/user-guide/data-sharing-intro より*

少し具体的にすると以下の流れで実現されます。

* Provider は共有するデータが入った Database を用意する
* Provider は Share を作成し、必要な設定する
  * 共有先の Consumer の Snowflake アカウントを Share に設定
  * 共有対象の Database を設定し、権限設定し実際に共有するテーブルなどのオブジェクトを設定
* Consumer は Share された情報をインポートし、自分のアカウントに **Read-only** な DB を作成し、それを利用する

このように Provider のある Database をあたかも Consumer 側にも存在するかのように見せる形でデータを共有します。

また、Snowflake のサービスレイヤとメタデータストアの情報をうまく共有することで、Provider と Consumer のアカウント間で**データのコピー・転送はされない**というのもポイントです。
Snowflake では Storage と Compute が分離されていますが、その恩恵を Data Sharing でも受けられるわけですね。

このありがたさは Data Sharing が無かった時代・使えない環境でデータ共有する方法を考えるとより分かりやすいかもしれません。この Data Sharing の機能を使わない場合には、 Provider と Consumer の権限を調整した上でデータ共有用の ETL パイプラインを作成し、その運用・監視をしないといけないわけです。 Snowflake で Data Sharing する場合にはこの辺りのパイプラインは全て不要で Share にまつわる設定だけ適切に管理すれば簡単にデータの共有ができてしまいます。



## Data Sharing のコスト

Data Sharing の仕組みを踏まえると比較的想像しやすいかと思いますが、 Consumer と Provider とでそれぞれ以下のようなコストがかかります。

* Consumer 側
  * データはコピー・転送されないためストレージコストはかからない
  * 共有されたデータを利用するためのウェアハウスを動かしたらその分の料金**のみ**がかかる
* Provider 側
  * データはプロバイダーの Database に実態が存在するため、ストレージ費用は Provider 側
  * クロスリージョン・クラウドに共有するために Listing の Auto-fulfill を利用したり、自分で Replication の仕組みを構築した場合その転送料金はかかる


## Share に関連する SQL

### DDL

DDL を見ておくと全体像が掴みやす句なると思います。
Share 自体はどのアカウントに共有をするか、という情報を保持するわけですね。

* [CREATE SHARE]( https://docs.snowflake.com/ja/sql-reference/sql/create-share )
  ```sql
  CREATE [ OR REPLACE ] SHARE [ IF NOT EXISTS ] <name>
    [ COMMENT = '<string_literal>' ]
  ```
* [ALTER SHARE]( https://docs.snowflake.com/ja/sql-reference/sql/alter-share )
  ```sql
  ALTER SHARE [ IF EXISTS ] <name> { ADD | REMOVE } ACCOUNTS = <consumer_account> [ , <consumer_account> , ... ]
                                          [ SHARE_RESTRICTIONS = { TRUE | FALSE } ]

  ALTER SHARE [ IF EXISTS ] <name> SET { [ ACCOUNTS = <consumer_account> [ , <consumer_account> ... ] ]
                                        [ COMMENT = '<string_literal>' ] }

  ALTER SHARE [ IF EXISTS ] <name> SET TAG <tag_name> = '<tag_value>' [ , <tag_name> = '<tag_value>' ... ]

  ALTER SHARE <name> UNSET TAG <tag_name> [ , <tag_name> ... ]

  ALTER SHARE [ IF EXISTS ] <name> UNSET COMMENT
  ```


### DCL

どのデータを共有するかは Share に対する権限付与で行います。詳細は後述します。
* [GRANT <privilege> … TO SHARE]( https://docs.snowflake.com/en/sql-reference/sql/grant-privilege-share )
  ```sql
  GRANT objectPrivilege ON
      {  DATABASE <name>
        | SCHEMA <name>
        | FUNCTION <name>
        | { TABLE <name> | ALL TABLES IN SCHEMA <schema_name> }
        | { EXTERNAL TABLE <name> | ALL EXTERNAL TABLES IN SCHEMA <schema_name> }
        | { ICEBERG TABLE <name> | ALL ICEBERG TABLES IN SCHEMA <schema_name> }
        | { DYNAMIC TABLE <name> | ALL DYNAMIC TABLES IN SCHEMA <schema_name> }
        | TAG <name>
        | VIEW <name>  }
    TO SHARE <share_name>
  ```
* [GRANT DATABASE ROLE … TO SHARE]( https://docs.snowflake.com/en/sql-reference/sql/grant-database-role-share )
  ```sql
  GRANT DATABASE ROLE <name>
    TO SHARE <share_name>
  ```
  * Database role への権限追加は https://docs.snowflake.com/en/sql-reference/sql/grant-privilege を見てください


## Data Sharing 可能なオブジェクト

以下のオブジェクトが共有可能です。

* Database
* Tables
* Dynamic tables
* External tables
* Iceberg tables
* Secure views
* Secure materialized views
* Secure UDFs

### secure vs. non-secure

基本的にデータ共有可能なのは Secure views ですが、普通の view とは何が違うのでしょうか？
端的に言うと non-secure view つまり普通の view だと、データが公開されてしまうリスクがあるからです。

* non-secure view ではその定義が他のユーザーに表示される
* non-secure view では内部最適化のためにベーステーブルのデータへアクセスする必要があり、間接的にデータが公開され得る

secure view ではその定義は所有者以外見ることができず、最適化のためにデータへのアクセスもしなくなります。

secure view を使うべきか、 non-secure view を使うべきかはプライバシーやセキュリティとクエリパフォーマンスのトレードオフを検証する必要があります。

materialized view でも UDF でも大体同じです。詳細は以下をご確認ください。

https://docs.snowflake.com/en/user-guide/views-secure
https://docs.snowflake.com/en/developer-guide/secure-udf-procedure

### non-secured views の共有

secure vs. non-secure で確認したように、プライバシーやセキュリティを優先し Data Sharing では基本的に Secure views は共有できますが、普通の view は共有できないようになっています。しかしパフォーマンスなどを優先するために普通の view を共有したいというユースケースもあるでしょう。

そういった場合には明示的に non-secure なオブジェクトの共有を許可することができます。
```sql
CREATE OR REPLACE SHARE allow_non_secure_views
 SECURE_OBJECTS_ONLY=FALSE
 COMMENT="Share views that require query optimization";
```
というように `SECURE_OBJECTS_ONLY=FALSE` と明示して Share を作成することで実現できます。

詳細は以下をご覧ください。
https://docs.snowflake.com/en/user-guide/data-sharing-views


## Data Sharing の方式

Snowflake で Data Sharing をする方法として、 Listing と Direct Share の２つがあります。

|         ポイント          |                                            Listing                                             |        Direct Share        |
| :-----------------------: | :--------------------------------------------------------------------------------------------: | :------------------------: |
|   共有対象のアカウント    |                                   任意のクラウド・リージョン                                   | 同一のクラウド・リージョン |
|       Auto-fulfill        |                                              対応                                              |           未対応           |
|   有料データオプション    |                                               有                                               |             無             |
|   データ公開オプション    |                                               有                                               |             無             |
| Consumer の利用メトリクス |                                               有                                               |             無             |
|      メタデータ提供       |                                               可                                               |            不可            |
|         Terraform         | 未対応 （[Issue](https://github.com/Snowflake-Labs/terraform-provider-snowflake/issues/2379)） |            対応            |


基本的には Listing の方が機能が多いですが、 Listing の中でも共有対象データを整理するために Share が使われるため、
本記事では Share に関する様々なことをまとめていきます。


## Share するオブジェクトの管理

Provider は Share に対し適宜権限を付与することで、 Consumer に共有するオブジェクトを管理することができます。
Share の権限管理の方法には

* オブジェクトの権限を直接 Share に Grant する
* Database role を用意しオブジェクトの権限を付与し、その Database role を Share に Grant する

の２つの方法があります。

### 権限を直接 Share に Grant する

以下の手順で実行します。

1. Share を作成する
   * サンプル
        ```sql
        CREATE SHARE test_share comment = 'test of data sharing';
        ```
   * この時点では `SHOW SHARES;` した時の `database_name` は空のまま
2. 共有対象の Database を Share に紐づける
   * サンプル
        ```sql
        GRANT USAGE ON DATABASE test_db TO SHARE test_share;
        ```
   * ここで `SHOW SHARES;` すると `database_name` に値が設定されているはずです
   * **Share が対象とする Database はあくまで一つなため、一つの DB の USAGE 権限しか付与できません**
   * 複数の Database の USAGE を付与しようとすると以下のようなエラーが出ます
        ```SQL
        GRANT USAGE ON DATABASE test_other_db TO SHARE test_share;
        -- 003033 (0A000): SQL compilation error:
        -- Database 'TEST_OTHER_DB' does not belong to the database that is being shared.
        ```
   * この辺りは複数のDBを参照する View などを共有したい場合に引っかかりやすいので、後述の「複数 DB のデータの共有」も見てください
3. 2 で追加した Database 内の対象の Shcema も Usage 権限を付与する
   * サンプル
        ```sql
        GRANT USAGE ON SCHEMA test_db.sch TO SHARE test_share;
        ```
   * 2 で設定した DB 内の Schema であれば、複数の Schema に対し Usage 権限を付与しても良い
4. 3 で設定した Schema 内のオブジェクトの各種権限を付与する
   * テーブルやビューなど、オブジェクトに応じて権限を付与すればOK
   * オブジェクトや権限によって様々な注意があり、代表的なものは以下
     * 基本的に view に対する select 権限は secure view にのみ可能で、 non-secure view に対して実行するとエラーになる
     * 基本的に UDFs に対する usage 権限は secure UDFs にのみ可能で、 non-secure UDFs に対して実行するとエラーになる
     * Future grants はサポートされておらず、新しいオブジェクトに対しては都度権限を付与する必要がある


Share への権限の付与については以下のドキュメントが詳しいです。特に "Usage notes" にはよくある落とし穴が記載されているのでぜひご覧ください。
https://docs.snowflake.com/en/sql-reference/sql/grant-privilege-share

### Database role を Share に Grant する

以下の手順で実行します。

1. 共有対象の Database で Database role を作成する
   * Database role については以下が分かりやすいです
     https://zenn.dev/dataheroes/articles/snowflake-database-role-20240727
2. 1 の Database role にオブジェクトの権限を付与する
   * 詳細は https://docs.snowflake.com/en/sql-reference/sql/grant-privilege
   * いくつか注意点があります
     * 複数の Database を参照するビューなどのオブジェクトを共有したい場合には database role は利用できません
       * Database への `REFERENCE_USAGE` 権限の付与ができないからです
     * Future grants を利用している Database role は次の 4 で share と紐付けられません
       * 参考：https://docs.snowflake.com/en/release-notes/bcr-bundles/2023_05/bcr-1144
3. Share を作成する
4. 共有対象の Database を Share に紐づける
5. 1/2 で用意した Database role を Share に紐づける
   * サンプル
        ```sql
        GRANT DATABASE ROLE d1.r1 TO SHARE share1;
        GRANT DATABASE ROLE d1.r2 TO SHARE share1;
        ```


### 直接Grant vs. Database Role

あとで細かく見ていく通り、いくつかのポイントがあり、ユースケースに応じて使い分ける必要があります。

* Consumer 側で Shared Database の権限管理をする必要があるか
  * 必要がある場合には Database role を Provider が用意しておく必要がある
  * 直接 Grant していると "IMPORTED PRIVILEGES" で全てのアクセス権限を一括で付与するかどうかしか決めることができない
* 複数の Database を参照する view などのオブジェクトを共有する必要があるか
  * 必要がある場合には直接 Grant するしかない
  * なぜならば複数 Database への "REFERENCE_USAGE" 権限が必要になるが、 database role ではこれを管理できないから



## Share の管理をするための権限

Provider 側で Share は誰が管理できるでしょうか？

基本的に ACCOUNTADMIN であればデフォルトでできます。
ACCOUNTADMIN でなくても適切に "CREATE SHARE" などの権限を付与することで、前節で説明した Share するオブジェクトの管理を行うことが可能です。
詳細は以下のドキュメントをご覧ください。

https://docs.snowflake.com/en/user-guide/security-access-privileges-shares

"CREATE SHARE" の権限を付与していくと、 Share の管理がより柔軟にはなります。
しかし各ロールが所有するオブジェクトを公開できてしまうということになるため、気をつけましょう。


個人的には Terraform で ACCOUNTADMIN で管理するのが良いのではないかなと考えています。


## Consumer 側の権限管理

Consumer 側で Share から Shared Database を作ることでその中のオブジェクトの読み取り専用で利用できるようになります。
この Consumer 側の権限管理は、 Provider 側のオブジェクト管理の仕方によってできることが異なります。

* Provider が 権限を直接 Share に Grant している場合
  * 単一の権限 "IMPORTED PRIVILEGES" しか利用することができません
  * この権限をロールに付与すると、共有されたデータベース内の全てのオブジェクトにアクセスできるようになり、細かい制御ができません
* Provider が Database role を Share に Grant している場合
  * 用意された Database role を Consumer も利用することができます
  * 細かい権限管理が Consumer 側でも直接必要な場合には Provider が Database role を事前に複数用意しておく必要があります


この詳細は https://docs.snowflake.com/ja/user-guide/data-sharing-gs#choosing-how-to-share-database-objects の途中で書かれています。


## 複数 DB のデータの共有

![data-sharing-multiple-databases](/images/articles/snowflake-share/data-sharing-multiple-databases.png)
*https://docs.snowflake.com/en/user-guide/data-sharing-multiple-db より*


複数の DB からデータを共有したい場合、例えば複数のDBのデータを利用して作られる view を共有したい場合には注意が必要です。

このように複数の Database を参照するオブジェクトを共有する場合には、 "REFERENCE_USAGE" という Database の権限を適切に Share に付与する必要があります。

> REFERENCE_USAGE
> あるオブジェクトが、**異なるデータベースにある別のオブジェクトを参照する際に**、オブジェクト（例: 共有内のセキュアビュー）の使用を有効化します。他のデータベースに対する権限を共有に付与します。データベースに対するこの権限は、どの種類のロールにも付与することはできません。詳細については、 GRANT <権限> ... TO SHARE および 複数データベースからのデータの共有 をご参照ください。
> https://docs.snowflake.com/ja/user-guide/security-access-control-privileges#database-privileges より


Database role では "REFERENCE_USAGE" は設定できないため、必然的に直接 Share に Grant する方式で権限管理しなければならないことになります。

前述した、直接 Share に Grant する方式の手順の途中で
```sql
GRANT REFERENCE_USAGE ON DATABASE database1 TO SHARE share1;
GRANT REFERENCE_USAGE ON DATABASE database2 TO SHARE share1;
```
を適宜追加する必要があるイメージです。

以下のドキュメントが具体例を交えて分かりやすく説明しているので、一度試してみることをお勧めします。

https://docs.snowflake.com/en/user-guide/data-sharing-multiple-db



## クロスリージョン・クラウドでの共有

Direct Share ではクロスリージョン・クラウドで共有することができません。
そのため Provider 側で適切なクラウド・リージョンで Snowflake アカウントを用意しつつ、 Provider アカウント間で Replication の設定をした上で、 Direct Share を Consumer と行う、という構成を取る必要があります。

![global-data-sharing-basic](/images/articles/snowflake-share/global-data-sharing-basic.png)
*https://docs.snowflake.com/en/user-guide/secure-data-sharing-across-regions-platforms より*

詳細は以下を参照してください。
https://docs.snowflake.com/en/user-guide/secure-data-sharing-across-regions-platforms


そもそも listing を利用していれば [Cross-Cloud Auto-fulfillment](https://other-docs.snowflake.com/en/collaboration/provider-listings-auto-fulfillment) の機能を利用することでこの辺りをマネージドにやってくれたりします。

どちらにせよ、クロスリージョン・クラウドに共有するためには Replication に関する費用が発生することは要注意です。



## Business Ciritcal Edition が Provider の場合

Provider のアカウントが Business Ciritcal Edition の場合には注意が必要で、 Consumer のアカウントの Edition によってはデフォルトでは共有ができない場合があります。
この場合、以下のように明示的に `SHARE_RESTRICTIONS=false` という設定をして共有の許可が必要です。この設定のためには `OVERRIDE SHARE RESTRICTIONS` 権限を持つロールを利用する必要があります。
```sql
use role accountadmin;
grant override share restrictions on account to role sysadmin;

use role sysadmin;
alter share my_share add accounts = consumerorg.consumeraccount SHARE_RESTRICTIONS=false;
```

詳しくは以下をご覧ください。
https://docs.snowflake.com/en/user-guide/data-sharing-provider#data-sharing-and-business-critical-accounts
https://docs.snowflake.com/en/user-guide/override_share_restrictions


注意点として、2025年2月現在 Terraform ではまだ `SHARE_RESTRICTIONS` の取り扱いが対応していません。
https://github.com/Snowflake-Labs/terraform-provider-snowflake/issues/630


## 終わりに

Share の世界も奥が深く、改めて各種ポイントについてまとめました。
これでも Listing についてはほとんど触れられておらず、 Data Sharing 全体だとさらにいろいろな話があります。

Data Sharing は便利な機能ですが、奥が深いですね。


