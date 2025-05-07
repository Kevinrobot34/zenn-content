---
title: "Snowflake のキャスト"
emoji: "🏯"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Snowflake", "SQL", "DataEngineering"]
published: false
publication_name: finatext
---

## はじめに

こんにちは！ナウキャストのデータエンジニアのけびんです。



## まとめ

* Snowflake におけるデータ型の変換には明示的キャストと暗黙的キャストがある。
* dateadd 関数に日付を表す文字列を入れると暗黙的に timestamp にキャストされてから適用されるっぽく(?)、 TIMESTAMP_NTZ が帰ってくるので要注意
* SQL においてもデータ型には気をつけよう


## キャストについて

あるデータ型の値を別のデータ型に変換することをキャストと言い、詳細は以下のドキュメントにまとまっています。
https://docs.snowflake.com/ja/sql-reference/data-type-conversion

明示的キャストと暗黙的キャストとあるのでそれぞれ見ていきましょう。

### Explicit Casting / 明示的キャスト

その名の通り、明示的にデータ型の変換を行う方法です。大きく分けて３つの方法があります。

* [CAST関数]( https://docs.snowflake.com/ja/sql-reference/functions/cast ) / [TRY_CAST関数]( https://docs.snowflake.com/ja/sql-reference/functions/try_cast )
* キャスト演算子 `::`
* [`TO_DATE`]( https://docs.snowflake.com/ja/sql-reference/functions/to_date ) や [`TRY_TO_DATE`]( https://docs.snowflake.com/ja/sql-reference/functions/try_to_date ) など、データ型ごとの変換関数

```sql
SELECT CAST('2022-04-01' AS DATE);
SELECT '2022-04-01'::DATE;
SELECT TO_DATE('2022-04-01');
```

変換関数については以下に詳細がまとまっています。
https://docs.snowflake.com/ja/sql-reference/functions-conversion


### Implicit Casting / 暗黙的キャスト

演算子や関数、 Stored Procedure に渡される値が想定されるデータ型と異なる場合に、強制的にキャストが行われることを暗黙的なキャストという。 **Coercion (強制) ** とも言ったりするらしい。

* 関数の例
  * テーブル my_table の my_integer_column 列のデータ型は integer
  * 関数 my_float_function は引数として float を受け取る
  * この際に、暗黙的に integer から float への変換が走る
    ```sql
    SELECT my_float_function(my_integer_column) FROM my_table;
    ```
* 演算子の例
  * `||` の演算子は文字列を受け取り連結する演算子
  * 以下のように左側に 17 という integer を持ってきているが、これは文字列に変換され、最終的に ‘1776’ という文字列になる
    ```sql
    SELECT 17 || '76';
    ```


### キャストの優先順位

以下の通りキャストは優先順位が高いようです。

```sql
-- キャスト演算子は算術演算子 * （乗算）よりも優先順位が高い
SELECT height * width::VARCHAR || " square meters" FROM dimensions;
SELECT height *(width::VARCHAR)|| " square meters" FROM dimensions; -- この意味になる

-- キャスト演算子は単項マイナス（否定）演算子よりも優先順位が高い
SELECT -0.0::FLOAT::BOOLEAN;
SELECT -(0.0::FLOAT::BOOLEAN); -- この意味になり、 bool にマイナスはつけられないのでエラーになる
SELECT (-0.0::FLOAT)::BOOLEAN; -- ではない
```

意外と勘違いしやすいかと思うので、複雑な式の場合には明示的にカッコでくくっておくのが大事になります。


### キャストできるデータ型

以下参照
https://docs.snowflake.com/ja/sql-reference/data-type-conversion#data-types-that-can-be-cast


## 文字列 → 日付・時刻の暗黙的キャスト

ここからが本題です。
dateadd 関数に日付を表す文字列を渡したら、date型で返して欲しいところですが実際はそうならない。
以下の通り、 TIMESTAMP_NTZ になってしまう。

```sql
select 
    '2025-05-07' as data_date_str,
    data_date_str::date as data_date,
    dateadd('year', -1, data_date_str),
    dateadd('year', -1, data_date),
;
```


元の列のデータ型が何なのかを意識しつつ開発するのが大事。 dbt sql model を書いているとデータ型の情報は見えにくかったりする。import の CTE でキャストが必要なくてもキャストの関数を書いておくことで、モデルの見通しが良くなるかも？と思ったりした。

```sql
with
  import_transactions as (
    select
      transaction_id::varchar as transaction_id,
      transaction_date::date as transaction_date,
      transaction_created_at::date as transaction_created_at,
      store_name_code::varchar as store_name_code,
      sales::integer as sales
    from {{ ref("raw_transaction") }}
  ),
...
```
