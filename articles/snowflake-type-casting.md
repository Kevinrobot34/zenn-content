---
title: "Snowflake のキャストと落とし穴"
emoji: "🏯"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Snowflake", "dbt", "SQL", "DataEngineering"]
published: false
publication_name: finatext
---

## はじめに

こんにちは！ナウキャストのデータエンジニアのけびんです。

日々 Snowflake と dbt を使ってパイプライン開発をしているのですが、その中で「日付を表す文字列を dateadd 関数に渡したら、タイムスタンプ型として戻されることに気づかず困った」という場面に直面しました。そこで今回のブログでは Snowflake のキャストについて簡単にまとめ、「文字列 → 日付・日時」の暗黙的キャストの落とし穴とその対策を紹介しようと思います。


## キャストについて

あるデータ型の値を別のデータ型に変換することをキャストと言い、詳細は以下のドキュメントにまとまっています。
https://docs.snowflake.com/ja/sql-reference/data-type-conversion

**明示的キャスト** と **暗黙的キャスト** があるのでそれぞれ見ていきましょう。

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

CAST関数とキャスト演算子 `::` は、内部的には `TO_DATE` など適切な変換関数を呼ぶようになっています。キャストの細かい挙動を知りたい場合には対応する変換の変換関数のドキュメントを見てみましょう。

変換関数については以下に詳細がまとまっています。
https://docs.snowflake.com/ja/sql-reference/functions-conversion


### Implicit Casting / 暗黙的キャスト

演算子や関数、 Stored Procedure に渡される値が想定されるデータ型と異なる場合に、強制的にキャストされるのが暗黙的キャストです。 **Coercion (強制)** とも言うようです。

* 関数の例
  * テーブル `my_table` の `my_integer_column` 列のデータ型は integer
  * 関数 `my_float_function` は引数として float を受け取る
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

以下の通りキャスト演算子は優先順位が高いようです。

```sql
-- キャスト演算子は算術演算子 * （乗算）よりも優先順位が高い
SELECT height * width::VARCHAR || " square meters" FROM dimensions;
SELECT height *(width::VARCHAR)|| " square meters" FROM dimensions; -- この意味になる

-- キャスト演算子は単項マイナス（否定）演算子よりも優先順位が高い
SELECT -0.0::FLOAT::BOOLEAN;
SELECT -(0.0::FLOAT::BOOLEAN); -- この意味になり、 bool にマイナスはつけられないのでエラーになる
SELECT (-0.0::FLOAT)::BOOLEAN; -- こうではない
```

意外と勘違いしやすいかと思うので、複雑な式の場合には明示的にカッコでくくっておいたり、変換関数で記載したりするなどの工夫が大事になります。


### キャストできるデータ型

ソースとなるデータ型によって、キャスト可能なデータ型が異なります。一部抜粋して紹介しますが、詳細は以下をご覧ください。

https://docs.snowflake.com/ja/sql-reference/data-type-conversion#data-types-that-can-be-cast

#### 数値系

| ソースデータ型 | ターゲットデータ型 | キャスト可能 | 強制可能 | 変換関数      |
| -------------- | ------------------ | ------------ | -------- | ------------- |
| FLOAT          |                    |              |          |               |
|                | BOOLEAN            | ✔            | ✔        | TO\_BOOLEAN   |
|                | NUMBER             | ✔            | ✔        | TO\_NUMBER    |
|                | VARCHAR            | ✔            | ✔        | TO\_VARCHAR   |
|                | VARIANT            | ✔            | ✔        | TO\_VARIANT   |
| NUMBER         |                    |              |          |               |
|                | BOOLEAN            | ✔            | ✔        | TO\_BOOLEAN   |
|                | FLOAT              | ✔            | ✔        | TO\_DOUBLE    |
|                | TIMESTAMP          | ✔            | ✔        | TO\_TIMESTAMP |
|                | VARCHAR            | ✔            | ✔        | TO\_VARCHAR   |
|                | VARIANT            | ✔            | ✔        | TO\_VARIANT   |

#### 日付系

| ソースデータ型 | ターゲットデータ型 | キャスト可能 | 強制可能 | 変換関数      |
| -------------- | ------------------ | ------------ | -------- | ------------- |
| DATE           |                    |              |          |               |
|                | TIMESTAMP          | ✔            | ✔        | TO\_TIMESTAMP |
|                | VARCHAR            | ✔            | ✔        | TO\_VARCHAR   |
|                | VARIANT            | ✔            | ❌        | TO\_VARIANT   |
| TIME           |                    |              |          |               |
|                | VARCHAR            | ✔            | ✔        | TO\_VARCHAR   |
|                | VARIANT            | ✔            | ❌        | TO\_VARIANT   |
| TIMESTAMP      |                    |              |          |               |
|                | DATE               | ✔            | ✔        | TO\_DATE      |
|                | TIME               | ✔            | ✔        | TO\_TIME      |
|                | VARCHAR            | ✔            | ✔        | TO\_VARCHAR   |
|                | VARIANT            | ✔            | ❌        | TO\_VARIANT   |


##### 文字列系

| ソースデータ型 | ターゲットデータ型 | キャスト可能 | 強制可能 | 変換関数      |
| -------------- | ------------------ | ------------ | -------- | ------------- |
| VARCHAR        |                    |              |          |               |
|                | BOOLEAN            | ✔            | ✔        | TO\_BOOLEAN   |
|                | DATE               | ✔            | ✔        | TO\_DATE      |
|                | FLOAT              | ✔            | ✔        | TO\_DOUBLE    |
|                | NUMBER             | ✔            | ✔        | TO\_NUMBER    |
|                | TIME               | ✔            | ✔        | TO\_TIME      |
|                | TIMESTAMP          | ✔            | ✔        | TO\_TIMESTAMP |
|                | VARIANT            | ✔            | ❌        | TO\_VARIANT   |


## 文字列 → 日付・日時の暗黙的キャスト

ここからが本題です。
先ほど確認したように、 VARCHAR 型は様々なデータ型へと変換が可能です。このように変換先の候補が多い場合に、「どのデータ型に暗黙的キャストされるのか？」を把握しておかないと困ることがおきます。具体例を見てみましょう。

日付や時刻のを操作する [`dateadd`]( https://docs.snowflake.com/ja/sql-reference/functions/dateadd ) 関数は DATE / TIME / TIMESTAMP のどれかを受け取って、指定された処理を行う関数です。また戻り値のデータ型は、基本的に引数の `<date_or_time_expr>` のデータ型に対応するように決定されます。
```sql
DATEADD( <date_or_time_part>, <value>, <date_or_time_expr> )
```

この第３引数の `date_or_time_expr` に文字列型の値を渡すと暗黙的キャストがされることになるわけですが、データ型は何に変換されるでしょうか？
例えば `'2025-05-07'` という文字列であれば、 DATE 型に暗黙的にキャストしてから `DATEADD` を実行し、 DATE 型の戻り値にして欲しい気がしますよね。
しかし実際にはそうはならず以下の通り TIMESTMP_NTZ 型の値が返されます。

```sql
select 
    '2025-05-07' as data_date_str,
    data_date_str::date as data_date,
    dateadd('year', -1, data_date_str),
    dateadd('year', -1, data_date),
;
```
![sample-query](/images/articles/snowflake-type-casting/sample-query.png)

つまり「日付を表す文字列を dateadd に渡すと、タイムスタンプとして戻される」わけです。しかし僕はこの挙動を知らず、dateaddした列をさらに別の文字列と比較するために varchar に変換したところ `00:00:00.000` という余計な文字列が付いてしまい比較の結果がおかしくなってしまっていました。
このように暗黙的キャストの挙動は時に想定外なことがあるので、常にデータ型を意識しておくことの重要性を再実感しました。


## データ型を意識するための対策

弊社では dbt を活用してパイプライン開発をしていますが、 dbt sql mdoel のコードには意外とデータ型の情報が出てこないことが多いです。
そうすると先ほどのような暗黙的キャストの落とし穴に気づきにくいと思います。対策は色々あり得るかなと思いますが、気軽にできることとして、 import の CTE でキャストが必要なくても明示的にキャストをしておくと良いかなと思ったりしています。 Python のタイプヒント的な感じです。

こうすることで、 import するテーブルにどのような列がありそれぞれのデータ型も可視化されるようになるので、モデルの可読性も上がりますし、先ほどのような暗黙的キャストによる事故も防ぎやすくなるのではないかと考えています。

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

要は「SQLを記述する際にもデータ型には気をつけよう」ということなのですが、他にも便利な Tips などあればぜひ教えてください！