## はじめに

こんにちは！ナウキャストのデータエンジニアのけびんです。

Snowflake はいろんなことをよしなにやってくれて、多くのユースケースでパフォーマンスも良いという素晴らしいサービスですが、データエンジニアとして高度なパフォーマンス最適化やコスト最適化をやっていくためには Snowflake のアーキテクチャや細かい仕組みについて理解しておくのが大事です。

今回の記事では最適化の際に大事な要素の一つであるキャッシュについて解説します。
キャッシュの仕組みを理解するためにも Snowflake のアーキテクチャのおさらいもし、実際にキャッシュが利用される実例も紹介していきます！

SnowPro Core でも頻出の分野かと思うのでぜひ読んでいただけたら嬉しいです！

紹介すること
* Snowflake のアーキテクチャのおさらい
* Snowflake の３つのキャッシュについて
* キャッシュが実際に使われる場面を確認する


## Snowflake のアーキテクチャおさらい

![snowflake-architecture](/images/articles/snowflake-cache/snowflake-architecture.png)
*https://medium.com/snowflake/snowflake-architecture-edition-pricing-overview-ed23f7b3dc6f より*


“**Multi-Cluster Shared Data Architecture**” と呼ばれる Snowflake のアーキテクチャは３つのレイヤーから構成されています。

* Cloud Service Layer
  * Snowflake の論文では、システムの「頭脳」と称されている
  * 認証・認可やセキュリティ、最適化までSnowflakeの様々なサービスが含まれる
  * FoundationDB という Key-Value Store があり、様々なメタデータが永続化されている
* Compute Layer / Query Processing Layer
  * Snowflake の論文では、システムの「筋肉」と称されている
  * 要は Warehouse のところで、クエリ実行を司る
  * Compute と Storage が分離されていて嬉しい
    * 簡単に Warehouse を Scale up も Scale out もできる
    * Warehouse は幾つでも作れる
  * コストのメインはここ
* Storage Layer / Database Storage Layer
  * S3などのオブジェクトストレージが利用されている
  * テーブルデータがマイクロパーティションとして列指向形式でよしなに保存されていたり、クエリの結果が保存されていたりする
  * Snowflake が全て管理しており、顧客は直接はアクセスできない
  * 直接存在を意識することはあまりないが、 $25.00 / TB コストがかかること、パーティションやクラスタリングの構造がどうなっているか想像することはあるよね


## Snowflake におけるクエリ実行とコスト

![snowflake-query-lifecycle](/images/articles/snowflake-cache/snowflake-query-lifecycle.png =500x)
*https://www.linkedin.com/pulse/query-lifecycle-snowflake-minzhen-yang-7mbfc/ より*

Snowflake でクエリは以下の流れで実行されます。

1. Query を Snowflake が受信する
2. Query Result Cache を確認し、存在したら直ちに結果を返す
3. Query を Planner や Optimizer が確認し処理する
4. Virtual Warehouse が実際にクエリを実行する
5. 計算結果を返す


Snowflake のメインの時間的・金銭的コストは 4 の Warehouse での実際の計算処理の部分であり、これを如何に減らせるかがパフォーマンス最適化・コスト最適化における重要なテーマになります。

キャッシュを利用することはパフォーマンス最適化・コスト最適化の一つの方法です。 Snowfalke では３種類のキャッシュが用意されており、それぞれ仕組みも対象としているデータも異なるのでそれぞれ見ていきましょう。

## Snowflake の３つのキャッシュ

Snowflake には３種類のキャッシュが用意されていて、それぞれ動くレイヤや対象とするデータ、有効期限などが異なります。

|          | Query Result Cache | Metadata Cache |     Warehouse Cache      |
| :------: | :----------------: | :------------: | :----------------------: |
|  レイヤ  |   Cloud Service    | Cloud Service  |         Compute          |
|   対象   |    Query Result    |    Metadata    |      micropartition      |
| 有効期限 |       24時間       |     永続的     | Warehouse が動いている間 |


それぞれを詳細に見ていきましょう。


### Query Result Cache

Query Result Cache は Cloud Service Layer でのキャッシュで、同一のクエリが投げられた場合には Warehouse で再計算せずに、以前の結果を再利用するというキャッシュです。クエリ実行フローの ② のところで結果が直ちに返されるため Warehouse を使わずに済む = コストが安くなる、ということになります。

詳細はこちら。
https://docs.snowflake.com/ja/user-guide/querying-persisted-results

* レイヤ：Cloud Service
* 対象：クエリの結果
* 有効期限：24時間
* 利用時の条件
  * パラメーター [`USE_CACHED_RESULT`]( https://docs.snowflake.com/ja/sql-reference/parameters#label-use-cached-result ) が `TRUE` となっていること
    * アカウント・ユーザー・セッションそれぞれで設定できるパラメーターなので注意
  * クエリが以前実行されたものと基本的に完全一致すること
    * 大文字小文字の違いや table alias の利用といった構文の違いなど、計算結果がたとえ同じになるものであっても、クエリの文字列の違いがあるだけで Query Result Cache は利用できなくなる
    * ただし、空白の違いは無視してくれたりもする
  * クエリが参照しているテーブルが変更されていないこと
  * RANDOM 関数など、実行のたびに結果が変わる関数が含まれていないこと
  * 外部関数や Hybrid table がクエリに含まれていないこと
  * などなど...
* 利用されていることの確認方法
  * Query Profile にて以下のような "Query Result Reuse" と表示されていること
    ![query-profile-qrc](/images/articles/snowflake-cache/query-profile-qrc.png =300x)
  * SELECT.dev を利用している場合には Warehouse ごとのパフォーマンスのページに Query Result Cache Usage Rate というグラフがあるのでそれを確認するのでもOK
* 有効なユースケース
  * 24時間以内に完全に同一なクエリを繰り返し投げるようなワークロードの場合に Query Result Cache はかなり有効であることがわかります
  * 例えば Web アプリケーションのバックエンドとして Snowflake を利用しているような場合です。ただし、基本的に同一なクエリでないと Query Result Cache は使えないので、これを踏まえた実装をする必要があります
    * よくない例： `select * from target_table where some_id = ‘hoge’` と `hoge` の値が毎回違うクエリをサーバーから投げる
    * 良い例：`select * from target_table` をSnowflakeに投げて、 some_idでの絞り込みはバックエンドサーバーで行う


:::message
Web アプリケーションのバックエンドでSnowflakeを使うのが最適なのか？という話はもちろんあります。ただ例えばSnowflakeでデータを加工しており、そのデータを可視化するようなアプリケーションのプロトタイプを作っている、という場面などではWeb アプリケーションのバックエンドとしてSnowflakeを使うのもありかなと思います。
:::



### Metadata Cache

Cloud Service Layer では様々なメタデータを保存・更新しており、これらのデータを利用するだけですむクエリについては Warehouse を動かさずに結果を得ることができます。

Metadata Cache について言及されている公式の資料はあまり多くないですが、以下などがあります。
https://www.snowflake.com/en/blog/how-foundationdb-powers-snowflake-metadata-forward/
https://www.snowflake.com/data-cloud-glossary/metadata/

* レイヤ：Cloud Service
* 対象：様々なメタデータ
    * テーブルの行数
    * カラムの min / max といった統計情報
    * ...
* 有効期限：24時間
* 利用時の条件
  * 収集されているメタデータのみで結果が得られるクエリであること
    * カラムの min/max といった統計情報については、対象とするカラムのデータ型によっても有無が変わったりする
      * 後述するように int 型では min/max の値を “Metadata Based Result” として Cloud Service Layer で取得し結果を返せるが、 varchar だと warehouse を動かして実際にデータを確認しないといけなかったりする
* 利用されていることの確認方法
  * Query Profile にて以下のような "“Metadata Based Result" と表示されていること
    ![query-profile-mc](/images/articles/snowflake-cache/query-profile-mc.png =300x)
* 有効なユースケース
  * Metadata cache は基本的に使える時にはSnowflakeがよしなに使ってくれるのであまり意識することはないが、上記のようにデータ型を変えればうまく Metadata Cache が使えるみたいなパターンがないかはユースケースに応じて考えてみると良いかもしれない。



### Warehouse Cache

Data Cache とも呼ばれたりするやつです。

Warehouse はクエリを実行する際に Storage layer にアクセスし必要なデータを取得することになります。
Storage Layer にアクセスしにいく分のオーバーヘッドがあるので、このデータを Warehouse 内ででキャッシュしておき適宜再利用することでパフォーマンス向上を目指すのが Warehouse Cache です。

キャッシュ管理のアルゴリズムは LRU (Least Recently Used) なシンプルなもので、新しいデータをキャッシュに追加する前には最も利用されていないデータを削除することで、よく使われるデータがキャッシュとして保存されるようになっています。
この辺りは Snowflake の論文 “The Snowflake Elastic Data Warehouse” の “3.2.2 Local Caching and File Stealing” で述べられています。

その他の詳細はこちら。
https://docs.snowflake.com/ja/user-guide/performance-query-warehouse-cache


* レイヤ：Compute Layer
* 対象：マイクロパーティション（テーブルデータまたはその一部）
* 有効期限：Warehouseが起動している間
* 利用時の条件
  * **あとでちゃんと書く**
* 利用されていることの確認方法
  * Query Profile の Statistics で "Percentage scanned from cache" を見れば良い
    ![query-profile-wc](/images/articles/snowflake-cache/query-profile-wc.png =300x)
* 有効なユースケース
  * 仕組みから分かる通り、 Warehouse 側にデータを一時的に保存する形になるので、 Warehouse が auto suspend されるとその度にこのキャッシュはドロップされてしまう。
  * つまり Warehouse を auto suspend せず起動したままにした方が Warehouse cache の観点では良いが、一方でその分 Warehouse 自体の課金は発生してしまう。コストの観点ではこのトレードオフに注意してユースケースごとに Warehouse の auto suspend の時間を調整することが重要になります。
  * BI ツールなどから使っている Warehouse の場合、このキャッシュを維持するために auto suspend を10分にすると良いとドキュメントでは推奨している

Warehouse cache を考えるかどうかは、以下のようなクエリで ACCOUNT_USAGE.QUERY_HISTORY の “percentage_scanned_from_cache” を確認して見るのが大事。
```sql
SELECT warehouse_name
  ,COUNT(*) AS query_count
  ,SUM(bytes_scanned) AS bytes_scanned
  ,SUM(bytes_scanned*percentage_scanned_from_cache) AS bytes_scanned_from_cache
  ,SUM(bytes_scanned*percentage_scanned_from_cache) / SUM(bytes_scanned) AS percent_scanned_from_cache
FROM snowflake.account_usage.query_history
WHERE start_time >= dateadd(month,-1,current_timestamp())
  AND bytes_scanned > 0
GROUP BY 1
ORDER BY 5;
```


## Cache が使われていることを確認してみる

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
Snowflake のパフォーマンス検証などの際に `USE_CACHED_RESULT` を False としたいことがあると思いますが、このような場合には `ALTER SESSION SET USE_CACHED_RESULT = FALSE;` とセッションレベルで Query Result Cache を無効化するようにしましょう。
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

**Percentage scanned from cache が0→100%となるのをみせる**

### Metadata Cache の確認

**int / varchar でキャッシュが使われたりそうでなかったりする例を出す。**

### `USE_CACHED_RESULT` をオンに戻す

Query Result Cache を確認するために `USE_CACHED_RESULT` を元に戻しましょう
```sql
ALTER SESSION SET USE_CACHED_RESULT = TRUE;
SHOW PARAMETERS like '%CACHE%';
```

### Query Result Cache の確認

Query Profile が変わる事を示す。
Query Details も変わる事を示す。


## 終わりに

Snowflake の Cache には３種類あり、それぞれの特徴を見てきました。
Snowflake のアーキテクチャを踏まえそれぞれの役割があり非常に興味深いですよね。

基本的にはどの Cache も使える時にはよしなに使ってくれているため、普段はあまり意識しないかもしれないですが、
しっかり Cache を使って最適化を進めようと思うと、裏側の仕組みをイメージできることが大事だなと改めて思いました！

Cache を使いこなして良い Snowflake ライフを！
