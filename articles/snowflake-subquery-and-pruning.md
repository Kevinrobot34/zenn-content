---
title: "Snowflakeでサブクエリ利用時にプルーニングが効かない問題の理解と解決"
emoji: "❄️"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Snowflake", "dbt", "optimization", "DataEngineering"]
published: false
publication_name: finatext
---


## はじめに

こんにちは！ナウキャストのデータエンジニアのけびんです。

Snowflake での select のクエリを最適化するためには「不要なファイル（partition）を読み込まないようにする」、つまり **"partition pruning"** をいかに活用できるかが大事なポイントの一つです。

実は where 句の条件でサブクエリを利用しているとプルーニングが効かないという事象が知られています。本記事ではこの現象がなぜ起きるのかを解説するとともに、対策方法を２つ紹介します。


### まとめ

* クエリコンパイルは Cloud Service Layer で行われる
* Partition Pruning はクエリコンパイルの中で実行される
    * そのため Compile 時にフィルター条件が確定しないと Partition Pruning は実行できない
* 一般にサブクエリは計算しないと結果が分からないため、サブクエリを利用したフィルター条件を利用していると Partiton Pruning は実行できない
    * サブクエリの結果を変数に代入する形でクエリを分割し、フィルター条件ではその変数を参照する形にすると Pruning が効くようになる
* サブクエリを使った条件であっても Metadata Cache を利用できるようなサブクエリであればその結果はコンパイル時に分かるため、 Pruning は効く
    * Min/Maxを取るようなシンプルなクエリであってもデータ型によって Metadata Cache を利用できるかは変わる
    * 特に Varchar の場合 Metadata Cache が使えないため、日付を表す文字列は date 型で保存しておくのがおすすめ


## dbt incremental model とサブクエリ

まずはどういった場面で where 句の条件でサブクエリを利用していたのかを簡単に紹介します。

dbt incremental model では

```sql
select
    *
from {{ ref('app_data_events') }}
{% if is_incremental() %}
where data_date > (select max(data_date) from {{ this }} )
{% endif %}
```

というようなクエリを書くことがよくあります。

`(select max(data_date) from {{ this }} )` のサブクエリで対象のテーブルの最新の日付を取得し、元テーブルのうちそれよりも最新のデータのみ参照して差分更新する、というわけです。

差分更新なので使うべきデータは全体ではなく、最新のデータのみになります。そのためテーブルはフルスキャンではなく、  partition pruning を効かせ必要なデータだけを読み込めば良いはずです。


しかし以下の記事などに書かれているように、サブクエリを利用すると partition pruning が効かなくなってしまうことがあることが知られています。

> Not all predicate expressions can be used to prune. **For example, Snowflake does not prune micro-partitions based on a predicate with a subquery, even if the subquery results in a constant.**
> https://docs.snowflake.com/en/user-guide/tables-clustering-micropartitions#query-pruning


> This happens when the evaluation of a constant expression can not be performed at the compile time, in which the partition pruning is performed. This is due to the complex subquery against the clustered key TR_CALENDAR_D_KEY in the WHERE clause.
> [Snowflake Knowledge Base Articles - Pruning is not happening when using Subquery]( https://community.snowflake.com/s/article/Pruning-is-not-happening-subquery )


ここからは

* なぜサブクエリを利用すると partition pruning が効かなくなってしまうのか
* 逆に partition pruning を効かせるためにはどうすれば良いのか

を具体例を通して見ていきます。


## 検証データ・設定

まずは現象を再現するために検証用のデータを用意しましょう。以下のような POS データを模したテストデータを利用します。

| data_date  | store_id |        sales        |
| :--------: | :------: | :-----------------: |
| 2020-01-01 |    12    | 5288091600886742778 |
| 2020-01-01 |    28    | 3308066933530881735 |
|    ...     |   ...    |         ...         |

ある日付にある店舗でいくら売れたか、という簡単なトランザクションデータです。

このテーブルから、最新の日付の総売上を取得したい場合、こんな感じのクエリを書きたくなります。

```sql
select 
    data_date,
    sum(sales)
from pos_sample_1
where data_date = (select max(data_date) from pos_sample_1)
group by data_date;
```

dbt incremental model とは厳密には違いますが、「サブクエリでフィルタリングする」という点で構造は同じです。

このクエリは気持ち的には最新の日付が含まれる partition のみ読み込めば良いはずなので、 pruning が効いてほしいですよね。しかし実際には細かい条件によって pruning が効いたり効かなかったりします。


### pruning が効かない状況を確認

**`data_date` 列を varchar で用意していると pruning が効きません。**
まずは実際にこれを確認しましょう。以下のクエリで `pos_sample_1` というテーブルを用意します。

```sql
create or replace table pos_sample_1 (data_date varchar, store_id int, sales int) as
select 
    to_char(dateadd(day, trunc(seq4()/100000), '2020-01-01'::date)) as data_date, 
    seq4()%100000 as store_id, 
    abs(random(123)) as sales
from table(generator(rowcount => 100000*1000))
order by data_date, store_id;
```

これに対して以下のように最新の日付に対する総売上を求めるクエリを実行して Profile を見てみましょう。
```sql
select 
    data_date,
    sum(sales)
from pos_sample_1
where data_date = (select max(data_date) from pos_sample_1)
group by data_date;
```

すると、以下のように2つの step で実行されていることがわかります。
* Step1: サブクエリ `(select max(data_date) from pos_sample_1)` の部分
    * <img title='Screenshot 2024-10-26 at 8.55.28.png' alt='Screenshot 2024-10-26 at 8.55.28' src='/attachments/e81c2fbf-3065-45fa-bba8-1ce17da13716' width="400" data-meta='{"width":1102,"height":1262}'>
* Step2: 集計のクエリ
    * <img title='Screenshot 2024-10-26 at 9.16.01.png' alt='Screenshot 2024-10-26 at 9.16.01' src='/attachments/ffe5504e-1408-423d-89b9-3fd7b7e6202a' width="400" data-meta='{"width":1068,"height":1170}'>
    * `Filter [2]` のノードは以下の通り step1 の結果でフィルターする感じになっています
        * <img title='Screenshot 2024-10-26 at 9.17.01.png' alt='Screenshot 2024-10-26 at 9.17.01' src='/attachments/47da401b-7629-42d4-b0d6-9292ea51f6d3' width="400" data-meta='{"width":1112,"height":328}'>


この Profile を見てパフォーマンス的に問題になるのは以下の二点です。

* Step2 において partition pruning が効いていない
    * 64/64 全ての parittion がスキャンされていることがわかります
    * 今回のテストテーブルは小さいですが、大きいテーブルだとここで時間がかかることになります
* Step1 において集計処理がされていること
    * よく「Snowflakeではパーティションごとに最小値・最大値などのメタデータ・統計情報を収集している」と言いますが、テーブルの `data_date` の最大値を求めるというのはまさにこのメタデータを利用できそうです
     * しかし実際には集計の計算が動いてしまっています

今回の記事ではなぜこの二点が起きてしまっているかを理解し、それを踏まえどうすればこれを解決しクエリを最適化できるかを解説します。



## Pruning と Compile

まずはなぜ Pruning が効かなかったのか？を見ていきます。

### Snowflake のクエリコンパイル

Snowflake がクエリを実行する流れを確認しましょう。

![snowflake-query-lifecycle](/images/articles/snowflake-subquery-and-pruning/snowflake-query-lifecycle.png =500x)
*https://www.linkedin.com/pulse/query-lifecycle-snowflake-minzhen-yang-7mbfc/ より*

* 1. Query を Snowflake が受信する
* 2. Query Result Cache を確認し、存在したら直ちに結果を返す
* 3. Query を Compiler や Planner、 Optimizer が確認し処理する
* 4. Virtual Warehouse が実際にクエリを実行する
* 5. 計算結果を返す

このようにクエリのコンパイルや最適化は Step3 で行われており、 Cloud Service Layer で実行されていることがわかります。

クエリのコンパイルの中身も見てみましょう。以下のような手順で実行計画が作られます。

![snowflake-compile-flow](/images/articles/snowflake-subquery-and-pruning/snowflake-compile-flow.png)
*https://youtu.be/CPWn1SZUZqE?si=vpirhGjKmnT-OCE9&t=1549 より*

これらの重要なポイントは

* クエリコンパイルは **Cloud Service Layer** で行われること
* コンパイルの一部として **"Micro Partition Pruning"** が含まれる

という点です。

つまり、 partition pruning を行うためには **Cloud Service Layer でコンパイル時（実際の計算前）にそれに足る情報が得られる必要がある** ということになります。


### 実験：定数で条件指定

Pruning と Compile の関係を踏まえると、「サブクエリを使っていたため、クエリコンパイル時にはフィルター条件が定まらず、Partition Pruningを行えていなかった」と理解できます。

実験として、最新の日付をリテラルで指定してみましょう。

```sql
select 
    data_date,
    sum(sales)
from pos_sample_1
where data_date = '2022-09-26'
group by data_date;
```

このようにすることで、クエリコンパイル時に pruning する条件が分かるようになるため、partition pruning が効くことになります。実際に Profile で以下の通り確認できます。

<img title='Screenshot 2024-10-26 at 9.44.00.png' alt='Screenshot 2024-10-26 at 9.44.00' src='/attachments/a89136d5-7759-4320-9c3b-aedb71ecd3b6' width="400" data-meta='{"width":1142,"height":1022}'>



### 解決策①：クエリの分割

直接リテラルで指定できれば苦労はしません。毎回テーブルの最大の日付で動的に絞り込みたい、というケースは多々あるでしょう。
Partition Pruning のためにはサブクエリを使わなければ良いだけなので、クエリを分割しましょう。

具体的にはサブクエリの結果を一旦変数に代入し、それを where 句で利用すれば良いです。こうすることで解決策①と同様な理由により Pruning が効くようになります。

```sql
set data_date_max = (select max(data_date) from pos_sample_1);

select 
    data_date,
    sum(sales)
from pos_sample_1
where data_date = $data_date_max
group by data_date;
```



## Metadata Cache

そもそもサブクエリとして実行していた、 `select max(data_date) from pos_sample_1` は単なるカラムの最大値を求めるシンプルなものでした。

これは Cloud Service Layer で収集されているメタデータから取得できそうに思えます。そう考えるとコンパイル時にこのサブクエリの結果は分かってもおかしくないはずで、 partition pruning が効かなかったのも不思議に感じてきます。

実はこれは Snowflake におけるキャッシュの一つである Metadata Cache の仕様によるものです。


### Snowflake の Metadata Cache おさらい

Snowflake は Cloud Service Layer 
データ型によって "METADATA-BASED RESULT" で cloud service layer で返せるかどうかが変わる。

https://zenn.dev/finatext/articles/snowflake-chache

```sql
create or replace table cache_test (id int, code varchar, name varchar) as
select seq8(), 'code-' || seq4()%100, randstr(10, random())
from table(generator(rowcount => 1000000));
```

この時、 Int 型の `id` 列の最小値・最大値を求める以下のクエリは、 "METADATA-BASED RESULT" となり、 Metadata Cache が使われていることがわかる。つまり Cloud Service Layer でメタデータのみを利用し warehouse を動かさなくても結果がわかる。
```sql
select min(id), max(id), count(*) from cache_test;
```
<img title='example-mc-1.png' alt='example-mc-1' src='/attachments/6c434f1b-2ac8-486e-92be-d2455d83aee0' width="400" data-meta='{"width":1484,"height":417}'>

しかし、 Varchar 型の `code` 列の最小値・最大値を取得するクエリを実行してみると、以下の通り集計のクエリが Warehouse で普通に実行されていることがわかる。
```sql
select min(code), max(code) from cache_test;
```
<img title='example-mc-2.png' alt='example-mc-2' src='/attachments/782c3f23-32ba-42cb-a5bf-7eb1748c9901' width="400" data-meta='{"width":1122,"height":1280}'>


このように統計情報から計算できそうなクエリでも、データ型によって挙動が変わる。
どうような検証を行うと NUMBER, BOOLEAN, DATE, TIME および各 TIMESTAMP 型ではメタデータを利用し結果を得られるが、 VARCHAR, BINARY, FLOAT型ではメタデータを利用できず集計処理が走ってしまうことがわかる。


### 解決策②：データ型を変更する

実は `pos_sample_1` テーブルでは `data_date` 列を varchar 型で定義されていました。
その結果、 `select max(data_date) from pos_sample_1` というサブクエリは "METADATA-BASE RESULT" は利用できず、毎回集計処理が必要だったというわけです。
そのためコンパイル時にはフィルター条件が定まらず Pruning ができなかったということになります。

そこで、 `data_date` 列を date 型で定義し直した `pos_sample_2` テーブルを作ってみましょう。

```sql
create or replace table pos_sample_2 (data_date date, store_id int, sales int) as
select 
    dateadd(day, trunc(seq4()/100000), '2020-01-01'::date) as data_date, 
    seq4()%100000 as store_id, 
    abs(random(123)) as sales
from table(generator(rowcount => 100000*1000))
order by data_date, store_id;
```

この新しいテーブルで同様なクエリを実行しましょう。
```sql
select 
    data_date,
    sum(sales)
from pos_sample_2
where data_date = (select max(data_date) from pos_sample_2)
group by data_date;
```
するとprofileは以下のようになっており、サブクエリを使っていても Pruning が効いていることが分かります。

<img title='Screenshot 2024-10-26 at 10.10.20.png' alt='Screenshot 2024-10-26 at 10.10.20' src='/attachments/fb871526-d438-4f75-9333-28b5a00f9903' width="400" data-meta='{"width":1126,"height":1090}'>


## まとめ


* クエリコンパイルは Cloud Service Layer で行われる
* Partition Pruning はクエリコンパイルの中で実行される
    * そのため Compile 時にフィルター条件が確定しないと Partition Pruning は実行できない
* 一般にサブクエリは計算しないと結果が分からないため、サブクエリを利用したフィルター条件を利用していると Partiton Pruning は実行できない
    * サブクエリの結果を変数に代入する形でクエリを分割し、フィルター条件ではその変数を参照する形にすると Pruning が効くようになる
* サブクエリを使った条件であっても Metadata Cache を利用できるようなサブクエリであればその結果はコンパイル時に分かるため、 Pruning は効く
    * Min/Maxを取るようなシンプルなクエリであってもデータ型によって Metadata Cache を利用できるかは変わる
    * 特に Varchar の場合 Metadata Cache が使えないため、日付を表す文字列は date 型で保存しておくのがおすすめ
