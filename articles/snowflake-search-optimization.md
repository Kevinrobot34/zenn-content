---
title: "Snowflake Search Optimization 徹底解説"
emoji: "🔍"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Snowflake", "DataEngineering", "SQL"]
published: false
publication_name: finatext
---

## はじめに

こんにちは！ナウキャストのデータエンジニアのけびんです。

Snowflake は様々な便利な機能がありますが、その一つに **"Search Optimization" （検索最適化サービス）** があります。 Search Optimization は大規模なテーブルに対する選択的な検索などのクエリのパフォーマンスを向上させるための仕組みです。

https://docs.snowflake.com/ja/user-guide/search-optimization-service


本ブログでは Search Optimization に関する概要を説明し、利用時の注意点や具体的なユースケースを紹介していきます。


## 仕組み

クエリのパフォーマンスを向上させるためには**余計なマイクロパーティションを読み込まない**ことが重要です。例えば Clustering ではデータをソートして似たデータが近いパーティションに含まれるようにすることで partition pruning の効率を向上します。今回のテーマである Search Optimization が有効化されると **"Search Access Path"** と呼ばれるデータ構造が追加で維持・管理されるようになります。特定のクエリでこの Search Access Path を参照することで選択的な検索などの際に partition pruning を可能にし、パフォーマンスの向上に役立てます。 

**"Search Access Path"** は **RDB における index** と似た働きをする、と表現するとイメージしやすい方も多いかもしれません。

![snowflake-arch](/images/articles/snowflake-search-optimization/how-it-works.png)
*https://quickstarts.snowflake.com/guide/getting_started_with_search_optimization/index.html#0 より*


Search Access Path のイメージとしては Python の Dict のような構造で、特定の列の値と Partition の対応関係を保持しているのだと思います。このようなデータを利用することで特定の検索クエリに対し参照すべき partition を高速に判定できるようになるわけです。
```python
search_access_path = {
    "Martin": ["Partition A"],
    "John": ["Partition B", "Partition D"],
    "Kevin": ["Partition C"],
    ...
} # <column value>: <list of partition id>
```

:::message
これはあくまでイメージです。実際の実装はもっと複雑でしょうし、後述する Search Method によっても保持するべきデータ構造は変わると思われます。「Dictみたな構造で列の値と Partition の対応関係を保持しておけば `O(1)` とかで参照すべき partition がわかるので partition pruning に使える」ということがキモです。
:::


## 用語

* Search Access Path
  * Search Optimization を有効化したテーブルで新しく維持管理されるデータ構造のこと
  * 対象テーブルの列の特定の値がどのマイクロパーティションに含まれるか、という情報を保持している
* Search Method
  * Search Optimization がどのような検索クエリを対象とするかを指定する
  * `EQUALITY` / `SUBSTRING` / `FULL_TEXT` / `GEO` の4つがあります
  * それぞれに応じておそらく必要となるデータ構造が違うため、有効化の際に指定する必要がある
  * 詳細は[こちら]( https://docs.snowflake.com/en/sql-reference/sql/alter-table#label-alter-table-searchoptimizationaction-add )をご覧ください


## Search Optimization の操作

以下で記載の通り基本的には以下のように Alter Table で設定を行います。
https://docs.snowflake.com/en/user-guide/search-optimization/enabling

```sql
-- Search optimization の有効化
ALTER TABLE t1 ADD SEARCH OPTIMIZATION ON EQUALITY(c1, c2);
-- Search Optimization の削除
ALTER TABLE t1 DROP SEARCH OPTIMIZATION ON EQUALITY(c1, c2);

-- Equality で可能な全ての列に対して有効化
-- コストが必要以上にかかり得るので個人的には非推奨
ALTER TABLE test_table ADD SEARCH OPTIMIZATION;

-- 以下のクエリであるテーブルで現在設定されている Search optimization を一覧することができます
DESCRIBE SEARCH OPTIMIZATION ON sample_data;
```

この際の `EQUALITY` が Search Method です。以下のとおり４種類あります。
* `EQUALITY`
  * `=` もしくは `in` による where 句があるクエリの最適化の際に使えます
  * ただし `not in` の場合には使えません
  * Join の際にも効果を発揮してくれることがあります
  * カラムの型としては numerical / string / binary / VARIANT と幅広く使えます
  * 詳細は[こちら](https://docs.snowflake.com/en/user-guide/search-optimization/point-lookup-queries)と[こちら](https://docs.snowflake.com/en/user-guide/search-optimization/join-queries)
* `SUBSTRING`
  * その名の通り部分文字列に関する検索、具体的には `LIKE` や `REGEXP_LIKE` を利用した where 句があるクエリの最適化に使えます
  * カラムの型としては string / VARIANT で使えます
  * 詳細は[こちら](https://docs.snowflake.com/en/user-guide/search-optimization/substring-queries)
* `FULL_TEXT`
  * [`SEARCH` 関数](https://docs.snowflake.com/ja/user-guide/querying-with-search-functions#using-the-search-function)などによる全文検索を行うクエリの最適化に使えます
  * 部分文字列の検索と違い、空白などで文字列をトークンに分割し、一致するトークンが含まれるかを検索します
    * 日本語のトークナイザーはないので、英語やIPアドレスなど特定の用途でしか使え無さそうです
  * カラムの型としては string / VARIANT で使えます
  * 詳細は[こちら](https://docs.snowflake.com/en/user-guide/search-optimization/text-queries)
* `GEO`
  * GEOGRAPHY 型のカラムに対して有効化するときに使います
  * 詳細は[こちら](https://docs.snowflake.com/en/user-guide/search-optimization/geospatial-queries)


## Search Optimization が有効なクエリ

Search Optimization は追加でデータ構造を管理・維持しておくことで特定のクエリのパフォーマンスを向上する機能です。追加のコストも発生するため、「どのようなクエリのパフォーマンスを向上できるのか？」を理解しておくことは非常に重要です。


### 一般に言えること

以下のようなクエリのパフォーマンス向上に役立つ可能性があります。

* 元々数秒以上の時間がかかっていたクエリ
  * 実行時間が1秒未満のクエリは Search Optimization を有効化しても大幅にパフォーマンスが向上することはない
* where 句で指定している列のカーディナリティが高い（10万以上の異なる値を持つ）こと
* where 句でフィルターした結果、参照すべきパーティションが少ないこと
  * search optimization は partition pruning できる機会を増やすという仕組みなので、元々多数の partition を読み込まないといけないクエリの高速化はできない
  * 言い換えると、ある列の各値が多くのパーティションに散らばって存在している場合、 search optimization を利用しても結局フルスキャンとあまり変わらない結果となってしまい効果が薄い
* クラスタリングキー以外での絞り込みを含むクエリ
  * クラスタリングキーと比較的相関があるカラムだと相性が良い


### データ型

* サポートされているデータ型
  * INTEGER / NUMERIC
  * DATE / TIME / TIMESTAMP
  * VARCHAR
  * BINARY
  * VARIANT / OBJECT / ARRAY
  * GEOGRAPHY
  * 先述したようにデータ型によって使える Search Method は異なるのでその点は要注意
* 逆にサポートされていないデータ型
  * **FLOAT と GEOMETRY**


### テーブル種別

Search Optimization は普通のテーブルに対して有効化可能です。一方、**外部テーブル・ビュー・マテリアライズドビューに対しては有効化できない**ので注意してください。



## 具体例での検証

### データの用意

法人マスターをイメージしたテーブルを以下のように作成します。

```sql
-- サンプルテーブル作成
CREATE OR REPLACE TABLE sample_data (
    id INT,
    corp_number STRING,
    company_name STRING,
    address STRING
);

-- サンプルデータの挿入（ランダムな13桁の法人番号）
INSERT INTO sample_data (id, corp_number, company_name, address)
SELECT
    ROW_NUMBER() OVER (ORDER BY seq4()),  
    LPAD(TO_CHAR(ABS(CAST(RANDOM() * 9999999999999 AS INT))), 13, '0'),  
    'Company_' || LPAD(TO_CHAR(ABS(CAST(RANDOM() * 9999999999999 AS INT))), 13, '0'),
    'Address_' || ROW_NUMBER() OVER (ORDER BY seq4())
FROM TABLE(GENERATOR(ROWCOUNT => 100000000));
```

実際のデータは以下のようなイメージです。
![sample-data](/images/articles/snowflake-search-optimization/sample-data.png)


### Search Optimization のコスト予測と有効化

システム関数の [SYSTEM$ESTIMATE_SEARCH_OPTIMIZATION_COSTS]( https://docs.snowflake.com/ja/sql-reference/functions/system_estimate_search_optimization_costs ) を利用することで、以下のように Search Optimization に関するコストの見積もりを行うことが可能です。


```sql
SELECT SYSTEM$ESTIMATE_SEARCH_OPTIMIZATION_COSTS('sample_data');
```

```json
{
  "tableName" : "SAMPLE_DATA",
  "searchOptimizationEnabled" : false,
  "costPositions" : [ {
    "name" : "BuildCosts",
    "costs" : {
      "value" : 0.029424,
      "unit" : "Credits"
    },
    "computationMethod" : "Estimated",
    "comment" : "estimated via sampling"
  }, {
    "name" : "StorageCosts",
    "costs" : {
      "value" : 0.002253,
      "unit" : "TB",
      "perTimeUnit" : "MONTH"
    },
    "computationMethod" : "Estimated",
    "comment" : "estimated via sampling"
  }, {
    "name" : "MaintenanceCosts",
    "computationMethod" : "NotAvailable",
    "comment" : "Insufficient data to compute estimate for maintenance cost. Table is too young. Requires 7 day(s) of history."
  } ]
}
```

ストレージコストとビルドにかかるコストがそれぞれ推定されています。テーブルによっては `MaintenanceCosts` で継続的にどれくらいの更新コストがかかるのかも見せてくれます。


### ポイントルックアップ

まずは `corp_number` 列に対するポイントルックアップを検証しましょう。

```sql
ALTER TABLE sample_data ADD SEARCH OPTIMIZATION ON EQUALITY(corp_number);
SHOW TABLES LIKE '%sample_data%';
-- search_optimization_progress の列が 100 になっていれば設定完了！
```

Search Optimization の設定が完了してから `corp_number` 列に対するポイントルックアップをしてみると、以下の通り **"Search Optimization Access"** となり、 partition pruning されていることが分かります。

```sql
select * from sample_data where corp_number = '4698799331488';
```

![profile1](/images/articles/snowflake-search-optimization/profile1.png =500x)

この際、何も設定していない `company_name` 列に対して同様なクエリを実行したり、 `corp_number` 列に対して部分文字列の検索を行ったりすると、フルスキャンになります。
```sql
select * from sample_data where corp_number like '%4698799331%';        -- EQUALITY(corp_number) なので部分文字列検索は非対応
select * from sample_data where company_name = 'Company_1820121784998'; -- EQUALITY(corp_number) なので company_name の検索には使えない
```
![profile2](/images/articles/snowflake-search-optimization/profile2.png =500x)
*どちらのクエリでもこのように80/80のすべてのパーティションをスキャンするような結果となる*


### 部分文字列・正規表現での検索

次に `company_name` 列での部分文字列検索ができるように設定してみましょう。

```sql
ALTER TABLE sample_data ADD SEARCH OPTIMIZATION ON SUBSTRING(company_name);
```

`company_name` 列に対して Like で部分文字列を含むかどうか検索すると、以下のように Search Optimization で pruning されます。

```sql
select * from sample_data where company_name like '%182012178%';
```

![profile3](/images/articles/snowflake-search-optimization/profile3.png =500x)


条件のところを変えてみるとよく分かりますが、 pruning できる量は条件によって変わります。場合によっては以下のように Search Optimization を使わずテーブルフルスキャンとなることもあります。

![profile4](/images/articles/snowflake-search-optimization/profile4.png =500x)
*"Search optimization service was not used because the cost was higher than a table scan for this query." と、SOSが使われていないクエリのプロファイルの例*


## 必要な権限

### 有効化時
Search Optimization は [`ALTER TABLE ... ADD SEARCH OPTIMIZATION`]( https://docs.snowflake.com/en/sql-reference/sql/alter-table#search-optimization-actions-searchoptimizationaction ) でテーブルに対し有効化しますが、設定時には以下の権限が必要です。

* テーブルに対する **OWNERSHIP** 権限
* テーブルを含む**スキーマ**に対する **ADD SEARCH OPTIMIZATION** 権限
  * おそらくスキーマの中で新たなデータ構造として search access path を作るため、 ADD SEARCH OPTIMIZATION 権限はスキーマの権限の一つなのだと思います


### 利用時

Search Optimization による最適化を行うためには特に追加の権限は必要ありません。 Search Optimization が有効化されたテーブルに対する SELECT 権限があれば、自動的に利用可能となっています。コンパイル時に必要と判断されれば Search Optimization が勝手に使われるだけという形です。



## コスト

Search Optimization を有効化するとコストが発生します。このコスト増加と Search Optimization によるコスト削減のトレードオフがあるので、 Search Optimization にかかるコストを理解し、適切に監視しながら利用することが大切になります。


### ストレージコスト

テーブルとは別に Search Access Path を追加で維持・管理することになるのでこの分のストレージコストが発生します。
極端な例で考えると、全ての列がユニークなテーブルに対して Search Optimization を有効化すると、 Search Access Path は下のテーブルと同等のサイズになる可能性があります。しかしドキュメントによると通常、サイズは元のテーブルの約1/4となるそうです。



### コンピューティングコスト

Search Optimization を有効化すると、 Search Access Path が構築・維持されますが、その際にコンピューティングコストが発生します。 Search Optimization はサーバレス機能の一つなため、この際に使われるのは Snowflake managed なコンピュートリソースです。

注意点としては以下のとおりです。
* 大きなテーブル（テラバイト（TB）以上のデータを含むテーブル）に検索最適化を追加すると、短期間にクレジット消費量が即座に増加する可能性があります
* テーブルのデータが大量に変更される場合にも Search Access Path の更新が必要となるためコストが追加でかかります
  * このあたりは clustering の費用とイメージは近いです
* 自動クラスタリングされているテーブルの場合、 Search Optimization のメンテナンスコストがさらに増加される可能性があります
  * パーティションが再構成されるので原理を考えると当たり前ではあります
  * テーブルのチャーンレートが高いほど、 Search Optimization のメンテナンスコストも高くなります
  * テーブル全体を再クラスタリングする場合には、一度 Search Optimization の設定を消し、再クラスタリングした後に、再度 Search Optimization を有効化した方が安いこともあるようです



### コストの見積もり

Search Optimization を実行する前に、そのコストを見積もっておくことは重要です。

[`SYSTEM$ESTIMATE_SEARCH_OPTIMIZATION_COSTS`]( https://docs.snowflake.com/ja/sql-reference/functions/system_estimate_search_optimization_costs ) 関数を利用することで、 Search Optimization 有効化時のコストを推定できます。


### コストの監視

以下のように Snowsight から確認することができます。

![snowsight](/images/articles/snowflake-search-optimization/snowsight.png)





## Search Optimization とその他の機能の比較

Search Optimization 以外にもパフォーマンスの向上に役立つ機能が Snowflake にはあります。
以下ページで、４つの機能の比較がされています。
* Search Optimization
* Query Acceleration
* Materialized View
* Clustering


https://docs.snowflake.com/ja/user-guide/search-optimization-service#other-options-for-optimizing-query-performance


## Search Optimization と Clustering

Search Optimization と Clustering は併用することが可能です。しかしコストもかかるので、以下のようなそれぞれの特性を活かした併用の仕方を模索するとコスパ良く検索クエリを捌けるようにできるはずです。

* Equality による pruning は Clustering によっても実現可能
  * そのため Clustering によるキーに対する Search Optimization は意味がない
  * それら以外の列での検索がある場合には追加で Search Optimization を検討の余地がある
  * また、 Equality 以外での検索の高速化をしたい場合には Search Optimization でないといけない
* コストについて
  * 自動クラスタリングされているテーブルの場合、 Search Optimization のメンテナンスコストがさらに増加される可能性があります
    * パーティションが再構成されるので原理を考えると当たり前ではあります
    * テーブル全体を再クラスタリングする場合には、一度 Search Optimization の設定を消し、再クラスタリングした後に、再度 Search Optimization を有効化した方が安いこともあるようです



## ナウキャストで検討しているユースケース

### 法人番号と法人名

ナウキャストでは法人番号をキーとしたマスターデータの整備を行なっています。具体例で作成したサンプルデータはこれをイメージしていました。
法人に関する検索は、
* 法人番号は一致の検索
* 法人名は部分文字列の検索

が多いと考えられるので、法人番号についてはクラスタリングの設定を行い、法人名については部分文字列での Search Optimization を有効化しておくことでコスパ良く幅広い検索条件に対応できるのではないかなと考えています。


## 終わりに

RDB における index のような機能である Search Optimization について解説してきました。
ぜひ使いこなして Snowflake のテーブルへの検索クエリのパフォーマンスを上げていきましょう！！
