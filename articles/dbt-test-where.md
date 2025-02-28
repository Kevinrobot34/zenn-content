---
title: "dbt test を効率化するTips"
emoji: "🗺️"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["dbt", "DataEngineering", "SQL", "Snowflake"]
published: false
publication_name: finatext

---

## はじめに

こんにちは！ナウキャストのデータエンジニアのけびんです。

現在ナウキャストでは dbt-snowflake を利用して Snowflake 上で ELT パイプラインを日々開発しています。その際に際にデータのクォリティの担保をするためには適切に dbt test を利用することが大事です。

今回の記事では効率の良い dbt test を書く方法を紹介します。


## 結論

dbt の generic test では where という config を利用することで、テストの SQL に絞り込み条件を追加することができ便利。

https://docs.getdbt.com/reference/resource-configs/where


## dbt の generic test おさらい

dbt test には自分で SQL でテストを書く Singular テストと、はじめから用意されており `schema.yml` に記述を少し追加するだけでテストを行える Generic テストがあります。

not-null 制約などよくある内容であれば、 Generic テストを利用するのが便利です。例えば弊社で取り扱っている POS データの transaction に関するデータであれば以下のような yaml を書いておくだけでカラムごとに適宜 dbt test が簡単に定義できます。（サンプルとして簡略化しています。）


```yaml
version: 2

models:
  - name: transaction_table
    description: transaction table of POS data
    columns:
      - name: store_code
        data_type: varchar(225)
        description: Unique code for each stores.
        tests:
          - dbt_expectations.expect_column_to_exist
          - dbt_expectations.expect_column_values_to_be_of_type:
              column_type: varchar
          - not_null
      - name: data_date
        data_type: date
        description: Date when this transaction record happened.
        tests:
          - dbt_expectations.expect_column_to_exist
          - dbt_expectations.expect_column_values_to_be_of_type:
              column_type: date
          - not_null
      - name: item_code
        data_type: varchar(225)
        description: Barcode identifier following JAN code regulation.
        tests:
          - dbt_expectations.expect_column_to_exist
          - dbt_expectations.expect_column_values_to_be_of_type:
              column_type: varchar
          - not_null
      - name: sales
        data_type: NUMBER
        description: Total amount in JPY indicating how much the item was sold at the date.
        tests:
          - dbt_expectations.expect_column_to_exist
          - dbt_expectations.expect_column_values_to_be_of_type:
              column_type: NUMBER
          - not_null
      - name: quantity
        data_type: NUMBER
        description: Total number of package indicating how much the item was sold at the date.
        tests:
          - dbt_expectations.expect_column_to_exist
          - dbt_expectations.expect_column_values_to_be_of_type:
              column_type: NUMBER
          - not_null
```

例えばこの `not_null` のテストは以下のようなクエリが実行されます。

```sql
select
    count(*) as failures,
    count(*) != 0 as should_warn,
    count(*) != 0 as should_error
from
    (
        select sales
        from db.pos.transaction_table
        where sales is null
    ) dbt_internal_test
```

この POS の transaction テーブル全体に対し、 not_null のテストが実行されていることがわかります。例えばこのテーブルが大きく、差分更新されているとすると、毎回全テーブルに対しテストを実行するのは勿体無いです。



## dbt test の where の config

dbt test には where という config を追加することができます。
https://docs.getdbt.com/reference/resource-configs/where

これを利用するとテストの途中に where 句を追加することができ、テストの効率化ができるわけです。実際に試してみましょう。以下のように `config` を追加してみます。

```yaml
version: 2

models:
  - name: transaction_table
    description: transaction table of POS data
    columns:
      ...
      - name: sales
        data_type: NUMBER
        description: Total amount in JPY indicating how much the item was sold at the date.
        tests:
          - dbt_expectations.expect_column_to_exist
          - dbt_expectations.expect_column_values_to_be_of_type:
              column_type: NUMBER
          - not_null
              config:
                where: "data_date = '{{ var('target_date') }}'"
      ...
```

すると、実行されるテストのクエリは以下のように変更されます。

```sql
select
    count(*) as failures,
    count(*) != 0 as should_warn,
    count(*) != 0 as should_error
from
    (
        select sales
        from (select * from db.pos.transaction_table where data_date = '2025-02-01') dbt_subquery -- ここが変更される
        where sales is null
    ) dbt_internal_test
```

シンプルですね。


便利なのですが注意点としては

* yaml ファイルの config は var / env_var は利用できる
* カスタムのマクロにはアクセスはできない

といった点になります。

あまり複雑になる config を追加するくらいなら Singular テストで対応するのが良いでしょう。ただ、ちょっとしたフィルタリングを追加するだけで良いような場合には非常に有効なのではないかなと思います。



## まとめ

dbt の Generic テストで使える where について紹介しました。
うまく使いこなせるとコスパ良くデータの品質を保てるようになると思うのでぜひお試しください！