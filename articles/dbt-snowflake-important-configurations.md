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

## Table


### Clustering 周りの設定

* `cluster_by`
  * デフォルト : none
  * これを設定しているとモデルの最後に order by 句が入る
* `automatic_clustering`
  * デフォルト : `false`
  * `alter {{ alter_prefix }} table {{relation}} resume recluster;` を実行するかどうかを制御する


### COPY GRANTS

dbt では run を実行すると `CREATE OR REPLACE TABLE AS SELECT ...` という CTAS のクエリが実行されます。２回目以降の dbt run では Replace の挙動となりますが、この際に、元のテーブルの権限を引き継ぐかどうかの設定が [`COPY GRANTS`]( https://docs.snowflake.com/ja/sql-reference/sql/create-table#label-create-table-copy-grants ) です。

デフォルトの設定だと `COPY GRANTS` が付与されないため、対象のテーブルの権限は毎回外れてしまいます。

https://github.com/dbt-labs/dbt-snowflake/blob/5d935eedbac8199e5fbf4022d291abfba8198608/dbt/include/snowflake/macros/relations/table/create.sql#L14
https://github.com/dbt-labs/dbt-snowflake/blob/5d935eedbac8199e5fbf4022d291abfba8198608/dbt/include/snowflake/macros/relations/table/create.sql#L43

以下のように `dbt_project.yml` で [`copy_grants`]( https://docs.getdbt.com/reference/resource-configs/snowflake-configs#copying-grants ) を `true` にしておくことでこれを回避できます。 

```dbt_project.yml
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