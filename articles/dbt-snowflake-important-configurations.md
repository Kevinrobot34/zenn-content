---
title: "dbt-snowflake の重要な設定"
emoji: "🦔"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["dbt", "DataEngineering", "SQL", "Snowflake"]
published: false
publication_name: finatext
---

## はじめに

こんにちは！ナウキャストのデータエンジニアのけびんです。

現在ナウキャストでは dbt-snowflake を利用して Snowflake 上で ELT パイプラインを日々開発しています。基本的には dbt の使い方を把握していれば開発は可能ですが、やはり Snowflake 特有の設定というのも存在します。本ブログでは dbt-snowflake で開発をする際に個人的に重要だと思う以下の設定についてまとめます。

* Table - Clustering
* Table - COPY GRANTS
* External Table
* Secure View


## Table - Clustering

### Partitioning と Clustering のおさらい

大規模なテーブルにおいて Clustering の設定はパフォーマンスを向上させるために重要です。

**Partitioning** とは大規模なデータを小さく管理しやすいデータに分割して保存しておく手法です。特定のカラムの値に応じてファイルを分割するということが多いです。例えば日付のカラムで Partitioning しておくと、日付に関する条件があるクエリでは Partition Pruning が可能になります。

一方 **Clustering** とはあるテーブル全体のデータの分布に関する話で、特定のカラムでデータをソートしておくことでテーブル全体のデータの局所性を増す（似たデータが近くに配置されるようにする）という手法です。

Snowflake では Clustering (ソート) の設定が Partitioning に直結するため非常に重要です。

Snowflake ではデータは micro partition として数百MBごとに分割して保存されますが、その作られ方としてはデータが来た順番に適宜分割されるイメージです。ソートされていないままのデータの場合、 どの micro partition にも各カラムで幅広い値のデータが分散して保存されてしまいます。一方事前にデータをソートしてからテーブルを作成するようにすると、各 micro partition にはソートで使用した列については特定の値だけ含まれるというような状況が作られるわけです。

![clustering-sample](/images/articles/dbt-snowflake-important-configurations/clustering-sample.png =600x)
*https://select.dev/posts/snowflake-clustering より。created_at でソートされているため created_at では pruning しやすい形で分割されているが、それ以外のカラムではどの partition でも幅広い値が含まれ pruning はできない形になっていることがわかる。*


ちなみに Clustering するタイミングは大きく分けて２つあります。

* テーブルを作成する前に明示的に order by を指定しておく
* automatic clustering として Snowflake に定期的にソートしてテーブルを再作成してもらう
  * automatic clustering に関しては clustering 自体を実行する金額と、これによるコスト削減金額とを天秤にかけて実行するかどうかを考えることが大事です

この辺りのことは以下の記事が詳しいです。
https://select.dev/posts/snowflake-clustering


### Clustering に関する設定

実際に Clustering を行うためには、 dbt model の config において `cluster_by` と `automatic_clustering` の２つを適宜設定すればOKです。

* `cluster_by`
  * デフォルト : none
  * モデルの最後に order by 句を入れるかどうかを制御できる
  * また、以下のクエリを時効するかを制御する
    ```sql
    alter {{ alter_prefix }} table {{relation}} cluster by ({{cluster_by_string}});
    ```
* `automatic_clustering`
  * デフォルト : `false`
  * モデル実行後、以下のクエリを実行するかを制御する
    ```sql
    alter {{ alter_prefix }} table {{relation}} resume recluster;
    ```

```sql
{{
    config(
        materialized="incremental",
        incremental_strategy="delete+insert",
        cluster_by=["date", "dim_code"],
        automatic_clustering=true,
    )
}}

select
    ...
```

より細かい挙動を知りたい場合には dbt-snowflake の以下のあたりのコードを参照してください。

https://github.com/dbt-labs/dbt-snowflake/blob/5d935eedbac8199e5fbf4022d291abfba8198608/dbt/include/snowflake/macros/relations/table/create.sql#L12-L13

https://github.com/dbt-labs/dbt-snowflake/blob/5d935eedbac8199e5fbf4022d291abfba8198608/dbt/include/snowflake/macros/relations/table/create.sql#L45-L58


## Table - COPY GRANTS

dbt では run を実行すると `CREATE OR REPLACE TABLE AS SELECT ...` という CTAS のクエリが実行されます。２回目以降の dbt run では Replace の挙動となりますが、この際に、元のテーブルの権限を引き継ぐかどうかの設定が [`COPY GRANTS`]( https://docs.snowflake.com/ja/sql-reference/sql/create-table#label-create-table-copy-grants ) です。

デフォルトの設定だと `COPY GRANTS` が付与されないため、対象のテーブルの権限は毎回外れてしまいます。

https://github.com/dbt-labs/dbt-snowflake/blob/5d935eedbac8199e5fbf4022d291abfba8198608/dbt/include/snowflake/macros/relations/table/create.sql#L14
https://github.com/dbt-labs/dbt-snowflake/blob/5d935eedbac8199e5fbf4022d291abfba8198608/dbt/include/snowflake/macros/relations/table/create.sql#L43

以下のように `dbt_project.yml` で [`copy_grants`]( https://docs.getdbt.com/reference/resource-configs/snowflake-configs#copying-grants ) を `true` にしておくことでこれを回避できます。 

```yaml
models:
  +copy_grants: true
```

:::message
access role を適切に設計し、スキーマ単位で `future tables` の権限を適切に利用できるようにしておくことでも上記の問題は解決できます。このような Snowflake におけるロール構成に関しては以下の資料もご覧ください。
https://speakerdeck.com/kevinrobot34/privilege-and-cost-management-in-snowflake

ナウキャストでも基本的にはスキーマ単位の `future tables` に対応した access role を用意しそれを用いて権限管理しているのですが、とあるユースケースではスキーマ単位ではなく特定のテーブルに対する SELECT 権限を Terraform で管理していました。
その際に Terraform では特に何も変更をしていないのに以下の差分が定期的に発生しており、 `copy_grants` の設定の必要性に気づきました。

```diff
  # snowflake_grant_privileges_to_account_role.specific_table_select will be updated in-place
! resource "snowflake_grant_privileges_to_account_role" "specific_table_select" {
        id                = "..."
!       privileges        = [
+           "SELECT",
        ]
        # (5 unchanged attributes hidden)

        # (1 unchanged block hidden)
    }
```
:::



## External Table

dbt-external-tables というパッケージを利用すれば、 Snowflake の External Table を dbt の source として設定することも可能です。

https://github.com/dbt-labs/dbt-external-tables

このパッケージを利用することで source の yaml において external というセクションにて外部テーブルに関する情報（ステージやパーティションなど）を追記することができるようになります。

```yaml

version: 2

sources:
  - name: POS_A
    schema: POS_A
    description: Lake layer of POS_A data
    tables:
      - name: original_transaction
        description: >
          POS_A transaction data.
          Data is located in `s3://{pos_bucket}/nikkei/source/original/` with Hive partition.
        columns:
          - name: data_date
            ...
        external:
          file_format: "(TYPE=CSV COMPRESSION=NONE SKIP_HEADER=0 BINARY_FORMAT=UTF8 NULL_IF=())"
          location: '@{{target.database}}.POS_A.S3_POS_A_BUCKET/transaction/'
          partitions:
            - name: received_date_partition
              data_type: date
              expression: TO_DATE(split_part(split_part(metadata$filename, '/', 7), '_', 1), 'YYYYMMDD')
              description: >
                One of the partition columns.
                Corresponding `data_date`.
            - ...
```

外部テーブルに関しては以下の記事もご覧ください。
https://zenn.dev/dataheroes/articles/snowflake-external-table


## Secure View

Snowflake はデータ共有系の機能が強いですが、その際のセキュリティを担保するために Snowflake の View には Secure View と普通の View があります。具体的な違いとしては

* 普通の View
  * View の定義がオーナー以外でも確認可能
  * View にクエリした際、定義情報などを利用して最適化が実行される
* Secure View
  * View の定義はオーナーしか確認できず、安全
  * Secure View にクエリした際、定義情報などは利用できず一部の最適化ができない

といった点が挙げられます。

dbt-snowflake において Secure view を作成するためには [`secure`]( https://docs.getdbt.com/reference/resource-configs/snowflake-configs#secure-views ) という設定を true にするだけでOKです。

https://github.com/dbt-labs/dbt-snowflake/blob/5d935eedbac8199e5fbf4022d291abfba8198608/dbt/include/snowflake/macros/relations/view/create.sql#L1-L9



## まとめ

基本的には https://docs.getdbt.com/reference/resource-configs/snowflake-configs にある程度記載がある内容ではありますが、実際の dbt-snowflake のコードも見つつ整理すると具体的な挙動などの理解が深まるかなと思います。

dbt-snowflake 使い倒していきましょう！