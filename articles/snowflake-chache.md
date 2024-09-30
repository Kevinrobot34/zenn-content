---
title: "Snowflake の３種のキャッシュ徹底解説"
emoji: "♻️"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Snowflake", "DataEngineering", "SQL", "Cache"]
published: true
publication_name: finatext
---


## はじめに

こんにちは！ナウキャストのデータエンジニアのけびんです。

Snowflake はいろんなことをよしなにやってくれて、多くのユースケースでパフォーマンスも良いという素晴らしいサービスですが、データエンジニアとして高度なパフォーマンス最適化やコスト最適化をやっていくためには Snowflake のアーキテクチャや細かい仕組みについて理解しておくのが大事です。

今回の記事では最適化の際に大事な要素の一つであるキャッシュについて解説します。
キャッシュの仕組みを理解するためにも Snowflake のアーキテクチャのおさらいもし、実際にキャッシュが利用される実例も紹介していきます！


## Snowflake のアーキテクチャおさらい

![snowflake-architecture](/images/articles/snowflake-cache/snowflake-architecture.png)
*https://medium.com/snowflake/snowflake-architecture-edition-pricing-overview-ed23f7b3dc6f より*


“**Multi-Cluster Shared Data Architecture**” と呼ばれる Snowflake のアーキテクチャは３つのレイヤーから構成されています。

* **Cloud Service Layer**
  * Snowflake の論文では、システムの「頭脳」と称されている
  * 認証・認可やセキュリティ、最適化までSnowflakeの様々なサービスが含まれる
  * FoundationDB という Key-Value Store があり、様々なメタデータが永続化されている
* **Compute Layer / Query Processing Layer**
  * Snowflake の論文では、システムの「筋肉」と称されている
  * 要は Warehouse のところで、クエリ実行を司る
* **Storage Layer / Database Storage Layer**
  * S3などのオブジェクトストレージが利用されている
  * テーブルデータがマイクロパーティションとして列指向形式でよしなに保存されていたり、クエリの結果が保存されていたりする
  * Snowflake が全て管理しており、顧客は直接はアクセスできない


## Snowflake におけるクエリ実行とコスト

アーキテクチャと合わせてクエリ実行時の流れを把握しておくと、 Cache がどのように働いているのかが意識しやすいです。

![snowflake-query-lifecycle](/images/articles/snowflake-cache/snowflake-query-lifecycle.png =500x)
*https://www.linkedin.com/pulse/query-lifecycle-snowflake-minzhen-yang-7mbfc/ より*

1. Query を Snowflake が受信する
2. Query Result Cache を確認し、存在したら直ちに結果を返す
3. Query を Planner や Optimizer が確認し処理する
4. Virtual Warehouse が実際にクエリを実行する
5. 計算結果を返す


Snowflake のメインの時間的・金銭的コストがかかる処理は 4 の Warehouse で実際の計算する部分であり、これを如何に減らせるかがパフォーマンス最適化・コスト最適化における重要なテーマになります。


## Snowflake の３つのキャッシュ

キャッシュを利用することはパフォーマンス最適化・コスト最適化の一つの方法です。 Snowfalke では３種類のキャッシュが用意されており、それぞれ仕組みも対象としているデータも異なるのでそれぞれ見ていきましょう。

|          | Query Result Cache | Metadata Cache |     Warehouse Cache      |
| :------: | :----------------: | :------------: | :----------------------: |
|  レイヤ  |   Cloud Service    | Cloud Service  |         Compute          |
|   対象   |    Query Result    |    Metadata    |      micropartition      |
| 有効期限 |       24時間       |     永続的     | Warehouse が動いている間 |


### Query Result Cache

Query Result Cache は Cloud Service Layer でのキャッシュで、同一のクエリが投げられた場合には Warehouse で再計算せずに、以前の結果を再利用するというキャッシュです。クエリ実行フローの ② のところで結果が直ちに返されるため Warehouse を使わずに済む = コストが安くなる、ということになります。

詳細はこちら。
https://docs.snowflake.com/ja/user-guide/querying-persisted-results

* レイヤ：**Cloud Service**
* 対象：クエリの結果
* 有効期限：**24時間**
* 利用時の条件
  * パラメーター [`USE_CACHED_RESULT`]( https://docs.snowflake.com/ja/sql-reference/parameters#label-use-cached-result ) が `TRUE` となっていること
    * アカウント・ユーザー・セッションそれぞれで設定できるパラメーターなので注意
  * **クエリが以前実行されたものと基本的に一致すること**
    * 大文字小文字の違いや table alias の利用といった構文の違いなど、計算結果がたとえ同じになるものであっても、クエリの文字列の違いがあるだけで Query Result Cache は利用できなくなる
    * ただし、空白の違いは無視してくれたりもする
  * クエリが参照しているテーブルが変更されていないこと
  * RANDOM 関数など、実行のたびに結果が変わる関数が含まれていないこと
  * 外部関数や Hybrid table がクエリに含まれていないこと
  * などなど...
* 利用されていることの確認方法
  * Query Profile にて以下のような "**Query Result Reuse**" と表示されていること
    ![query-profile-qrc](/images/articles/snowflake-cache/query-profile-qrc.png =300x)
  * SELECT.dev を利用している場合には Warehouse ごとのパフォーマンスのページに Query Result Cache Usage Rate というグラフがあるのでそれを確認するのでもOK

Query Result Cache は上手に使うと効果が大きいですが、一方で上記の通り有効期限が短かったり、利用時の条件が意外といろいろとあります。これらを踏まえると、24時間以内に完全に同一なクエリを繰り返し投げるようなワークロードの場合に Query Result Cache はかなり有効であることがわかります。
例えば Web アプリケーションのバックエンドとして Snowflake を利用しているような場合です。実装を工夫して Snowflake に投げるクエリはなるべく同一になるようにするのがポイントになります。
* よくない例： `select * from target_table where some_id = ‘hoge’` と `hoge` の値が毎回違うクエリをバックエンドサーバーから投げる
* 良い例：`select * from target_table` をSnowflakeに投げて、 some_idでの絞り込みはバックエンドサーバーで行う


:::message
Web アプリケーションのバックエンドでSnowflakeを使うのが最適なのか？という話はもちろんあります。ただ例えばSnowflakeでデータを加工しており、そのデータを可視化するようなアプリケーションのプロトタイプを作っている、という場面などではWeb アプリケーションのバックエンドとしてSnowflakeを使うのもありかなと思います。
:::



### Metadata Cache

Cloud Service Layer では様々なメタデータを保存・更新しており、これらのデータを利用するだけですむクエリについては Warehouse を動かさずに結果を得ることができます。これらのメタデータは **FoundationDB** という Key-Value Store で永続化されています。

Metadata Cache について言及されている公式の資料はあまり多くないですが、以下などがあります。
https://www.snowflake.com/en/blog/how-foundationdb-powers-snowflake-metadata-forward/

* レイヤ：**Cloud Service**
* 対象：様々なメタデータ
    * テーブルの行数
    * カラムの min / max といった統計情報
    * などなど...
* 有効期限：**なし（FoundationDBでメタデータは永続化されている）**
* 利用時の条件
  * 収集されているメタデータのみで結果が得られるクエリであること
    * カラムの min/max といった統計情報については、対象とするカラムのデータ型によっても有無が変わったりする
* 利用されていることの確認方法
  * Query Profile にて以下のような "**Metadata Based Result**" と表示されていること
    ![query-profile-mc](/images/articles/snowflake-cache/query-profile-mc.png =300x)

Metadata cache は基本的に使える時にはSnowflakeがよしなに使ってくれるのであまり意識することは少ないかもしれません。ただし上記の通り統計情報を取るだけっぽいものでもデータ型によって Metadata Cache が使えたり使えなかったりするので、実際に Query Profile で確認しながら Metadata Cache が使えないか模索するのも大事かもしれません。



### Warehouse Cache

Data Cache とも呼ばれたりするやつです。
Warehouse はクエリを実行する際に Storage layer にアクセスし必要なデータを取得することになります。Storage Layer にアクセスしにいく分のオーバーヘッドがあるので、このデータを Warehouse 内ででキャッシュしておき適宜再利用することでパフォーマンス向上を目指すのが Warehouse Cache です。

キャッシュ管理のアルゴリズムは **LRU** (Least Recently Used) なシンプルなものです。新しいデータをキャッシュに追加する前には最も利用されていないデータを削除することで、よく使われるデータがキャッシュとして保存されるようになっています。
この辺りは Snowflake の論文 “The Snowflake Elastic Data Warehouse” の “3.2.2 Local Caching and File Stealing” で述べられています。

その他の詳細はこちら。
https://docs.snowflake.com/ja/user-guide/performance-query-warehouse-cache


* レイヤ：**Compute Layer**
* 対象：マイクロパーティション（テーブルデータまたはその一部）
* 有効期限：**Warehouseが起動している間**
* 利用時の条件
  * 特になし
    * あえて言うならLRUベースのキャッシュ管理なので、同一のデータが直近で利用されておりキャッシュに残っていること
* 利用されていることの確認方法
  * Query Profile の Statistics で "**Percentage scanned from cache**" を見れば良い
    ![query-profile-wc](/images/articles/snowflake-cache/query-profile-wc.png =300x)

仕組みから分かる通り、 Warehouse 側にデータを一時的に保存する形になるので、 Warehouse が auto suspend されるとその度にこのキャッシュはドロップされてしまいます。
つまり Warehouse を auto suspend せず起動したままにした方が Warehouse cache の観点では良いことになりますが、一方でその分 Warehouse 自体の課金は発生してしまいます。コストの観点ではこのトレードオフに注意してユースケースごとに Warehouse の auto suspend の時間を調整することが重要です。

BI ツールなどから使っている Warehouse の場合、 Warehouse Cache が有効なことが多くこれを維持するために auto suspend を10分にすると良いとドキュメントでは推奨していたりします。



## Cache が使われていることを確認してみる

ここまでで Snowflake の３種類のキャッシュについて整理してきたので、ここからは実際にキャッシュが使われる場面を具体的に確認していきましょう！

Snowsight で Query History みたり Details みたりしつつ、必要に応じて Profile に飛びながら詳細を確認するのがおすすめです。
![snowsight-worksheet](/images/articles/snowflake-cache/snowsight-worksheet.png)


### データを用意する

適当な DB / Schema で以下のようなテスト用のテーブル `cache_test` を用意しておきます。
```sql
use database workspace_kosaku_ono;
use schema workspace;

create or replace table cache_test (id int, code varchar, name varchar) as
select seq8(), 'code-' || seq4()%100, randstr(10, random())
from table(generator(rowcount => 1000000));
```

### `USE_CACHED_RESULT` をオフに

まずは Query Result Cache が有効化されているかどうか確認しましょう。
基本的にはアカウントレベルで true とデフォルト値のままになっているはずです。
```sql
SHOW PARAMETERS like '%CACHE%';
+-------------------+-------+---------+---------+-----------------------------------------------------------------------------------------------------------------------------------------+---------+
| key               | value | default | level   | description                                                                                                                             | type    |
|-------------------+-------+---------+---------+-----------------------------------------------------------------------------------------------------------------------------------------+---------|
| USE_CACHED_RESULT | true  | true    | ACCOUNT | If enabled, query results can be reused between successive invocations of the same query as long as the original result has not expired | BOOLEAN |
+-------------------+-------+---------+---------+-----------------------------------------------------------------------------------------------------------------------------------------+---------+
```

:::message
Snowflake のパフォーマンス検証などの際に `USE_CACHED_RESULT` を False としたいことがあると思いますが、このような場合には `ALTER SESSION SET USE_CACHED_RESULT = FALSE;` と**セッションレベル**で Query Result Cache を無効化するようにしましょう。
筆者は慣れていなかった頃にアカウントレベルでを無効化してしまっていたようで、それに気づかずにしばらくの間 Query Result Cache を意図せず使えていなかったということがありました。。。
:::

一旦セッションレベルで無効化しておきます。
```sql
ALTER SESSION SET USE_CACHED_RESULT = FALSE;
SHOW PARAMETERS like '%CACHE%';
+-------------------+-------+---------+---------+-----------------------------------------------------------------------------------------------------------------------------------------+---------+
| key               | value | default | level   | description                                                                                                                             | type    |
|-------------------+-------+---------+---------+-----------------------------------------------------------------------------------------------------------------------------------------+---------|
| USE_CACHED_RESULT | false | true    | SESSION | If enabled, query results can be reused between successive invocations of the same query as long as the original result has not expired | BOOLEAN |
+-------------------+-------+---------+---------+-----------------------------------------------------------------------------------------------------------------------------------------+---------+
```

### Warehouse Cache の確認

まずは Warehouse Cache が使われる様子を確認しましょう。
どんなクエリでも良いのですが、例えば以下のクエリを２回連続で実行してみましょう。
```sql
select code, count(*)
from cache_test
group by code
order by code;
```

２回実行した分の Query Profile を確認してみると、

* 1回目は "Percentage scanned from cache" は 0% で、 Processing に一定時間がかかっている
* 2回目は "Percentage scanned from cache" は 100% で、 TableScan に時間がほぼかかてtない

と、 Warehouse Cache が活用されることでパフォーマンスが上がっていることが確認できます。
![example-wc](/images/articles/snowflake-cache/example-wc.png)


### Metadata Cache の確認

次に Metadata Cache が使われている様子も確認しましょう。

試しに、 Int 型の `id` 列の最小値・最大値、そしてテーブルの行数を確認するクエリを実行してみると、 Query Profile は "METADATA-BASED RESULT" だけとなり、 Metadata Cache が使われていることがわかります。
```sql
select min(id), max(id), count(*) from cache_test;
```
![example-mc-1](/images/articles/snowflake-cache/example-mc-1.png)

しかし、 Varchar 型の `code` 列の最小値・最大値を取得するクエリを実行してみると、以下の通り集計のクエリが Warehouse で普通に実行されることがわかります。
```sql
select min(code), max(code) from cache_test;
```
![example-mc-2](/images/articles/snowflake-cache/example-mc-2.png =500x)

このようにデータ型によっても、取得されているメタデータや統計情報は異なるようです。
この辺りについては詳細が述べられているドキュメントもあまりないようなので、随時 Query Profile を確認しながら、 Metadata Cache が使える形にクエリを変更したりテーブルを変更したりできないかを検討するのが良さそうです。


### `USE_CACHED_RESULT` をオンに戻す

Query Result Cache を確認するために `USE_CACHED_RESULT` を元に戻しましょう
```sql
ALTER SESSION SET USE_CACHED_RESULT = TRUE;
SHOW PARAMETERS like '%CACHE%';
+-------------------+-------+---------+---------+-----------------------------------------------------------------------------------------------------------------------------------------+---------+
| key               | value | default | level   | description                                                                                                                             | type    |
|-------------------+-------+---------+---------+-----------------------------------------------------------------------------------------------------------------------------------------+---------|
| USE_CACHED_RESULT | true  | true    | SESSION | If enabled, query results can be reused between successive invocations of the same query as long as the original result has not expired | BOOLEAN |
+-------------------+-------+---------+---------+-----------------------------------------------------------------------------------------------------------------------------------------+---------+
```

### Query Result Cache の確認

最後に Query Result Cache が使われる様子を確認しましょう。
Warehouse Cache を利用した時と同様のクエリを再度、２回連続で実行しましょう。
```sql
select code, count(*)
from cache_test
group by code
order by code;
```
２回目に実行したクエリの Query Profile を確認すると、 "QUERY RESULT REUSE" だけとなっていることが分かります。

![example-qrc](/images/articles/snowflake-cache/example-qrc.png)


### その他の Cloud Service Layer のクエリの確認方法

Query Result Cache や Metadata Cache を利用したクエリは Warehouse を稼働させず Cloud Service Layer で結果を取得するため、以下のように Query Details を見た際に Warehouse の情報が記述されていない、という特徴があります。
言い換えると、 Query History view などにも Warehouse は null 

![query-details](/images/articles/snowflake-cache/query-details.png)


また、 SELECT.dev を利用している場合には、 Warehouse のページからも確認できます。
具体的には Performance タブにて、 "Include Cloud Services Only" にチェックを入れて "Query Result Cache Usage Rate" を確認することで、 Query Result Cache の利用率を確認することもできます。

![example-select-dev](/images/articles/snowflake-cache/example-select-dev.png)


## 終わりに

Snowflake の Cache には３種類あり、それぞれの特徴を見てきました。
Snowflake のアーキテクチャを踏まえそれぞれの役割があり非常に興味深いですよね。

SnowPro Core でも頻出な分野なので抑えておきたいところですね！

基本的にはどの Cache も使える時にはよしなに使ってくれているため、普段はあまり意識しないかもしれないですが、
しっかり Cache を使って最適化を進めようと思うと、裏側の仕組みをイメージできることが大事だなと改めて思いました！

Cache を使いこなして良い Snowflake ライフを！
