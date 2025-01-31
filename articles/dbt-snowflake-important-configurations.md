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

現在ナウキャストでは dbt-snowflake を利用して Snowflake 上で ELT パイプラインを日々開発しています。

Snowflake 特有な dbt の設定についてまとめようと思います。

## Table - Clustering

### Partitioning と Clustering のおsらい

大規模なテーブルにおいて Clustering の設定はパフォーマンスを向上させるために重要です。

* clustering はデータをソートしておき、局所性を増す（似たデータが近くに配置されるようにする）こと
* partitioning は大規模なデータを小さく管理しやすいデータに分割して保存しておくこと
  * 日付のカラムの値に応じてデータを分割しておく、といった設定が多い
  * これにより日付に関するフィルタリングの際に partition pruning が可能になる
* Snowflake では Clustering (ソート) の設定が partitioning に直結する
  * Snowflake では micro partition として数百MBごとにデータを分割して保存している
  * テーブルを作成する際に、事前にデータをソートしておけば、その分上から順番に micro partition が作られる → ソートで設定した列の値で partition しているような状況になる
  * ソートだけでなく、定期実行する場合には自然と clustering / partitioning される部分もある
* Snowflake の automatic clustering について
  * 定期的にデータが追加されたりすると、 clustering されていない状態になってしまったりする
  * 「テーブルを指定されたカラムでソートし直して再作成することで clustering された状態にしておく」という操作を定期的に自動で行ってくれるのが automatic clustering
  * 定期的に dbt model のフルリフレッシュをして作り直してくれるイメージで、この操作自体にお金がかかるため、使い方は要注意
* これらの話を踏まえると以下の二点が大事
  * dbt model でテーブルを作る際に、データはソートしておく → `cluster_by` の設定
  * automatic clustering が必要であればそれを明示しておく → `automatic_clustering` の設定
* 参考
  * https://select.dev/posts/snowflake-clustering


### Clustering に関する設定

dbt model の config において `cluster_by` と `automatic_clustering` の２つを適宜設定すればOKです。

* `cluster_by`
  * デフォルト : none
  * これを設定しているとモデルの最後に order by 句が入る
* `automatic_clustering`
  * デフォルト : `false`
  * `alter {{ alter_prefix }} table {{relation}} resume recluster;` を実行するかどうかを制御する

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


https://github.com/dbt-labs/dbt-snowflake/blob/5d935eedbac8199e5fbf4022d291abfba8198608/dbt/include/snowflake/macros/relations/table/create.sql#L12-L13

https://github.com/dbt-labs/dbt-snowflake/blob/5d935eedbac8199e5fbf4022d291abfba8198608/dbt/include/snowflake/macros/relations/table/create.sql#L45-L55



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

ナウキャストでも基本的にはスキーマ単位の `future tables` に対応した access role を用意しそれを用いて権限管理ているのですが、とあるユースケースではスキーマ単位ではなく特定のテーブルに対する SELECT 権限を Terraform で管理していました。
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


https://zenn.dev/dataheroes/articles/snowflake-external-table

## View

* `secure`

https://github.com/dbt-labs/dbt-snowflake/blob/5d935eedbac8199e5fbf4022d291abfba8198608/dbt/include/snowflake/macros/relations/view/create.sql#L1-L11




## Query Tag / Comment

`query_tag` の config をつかしておくと dbt が実行するクエリにタグをつけておくことができる。

この辺りは SELECT 社の dbt-snowflake-monitoring が非常に便利。

```dbt_project.yml
dispatch:
  - macro_namespace: dbt
    search_order:
      - nowcast_datahub_nikkei_dev
      - dbt_snowflake_query_tags
      - dbt
query-comment:
  comment: "{{ dbt_snowflake_query_tags.get_query_comment(node) }}"
  append: true # Snowflake removes prefixed comments.
```

* https://docs.getdbt.com/reference/project-configs/query-comment
* https://select.dev/posts/snowflake-query-tags
* https://github.com/get-select/dbt-snowflake-monitoring




## References

* https://docs.getdbt.com/reference/resource-configs/snowflake-configs
* 