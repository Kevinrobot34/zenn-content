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

クエリのパフォーマンスを向上させるためには**余計なマイクロパーティションを読み込まない**ことが重要です。例えば Clustering ではデータをソートして似たデータが近いパーティションに含まれるようにすることで partition pruning の効率を向上します。今回注目している Search Optimization が有効化されると "Search Access Path" と呼ばれるデータ構造が追加で維持・管理されるようになります。選択的な検索等の特定のクエリでこの Search Access Path を利用することで partition pruning を可能にし、パフォーマンスの向上に役立てます。

![snowflake-arch](/images/articles/snowflake-search-optimization/how-it-works.png)
*https://quickstarts.snowflake.com/guide/getting_started_with_search_optimization/index.html#0 より*


Search Access Path のイメージとしては Python の Dict のような構造で、特定の列の値と Partition の対応関係を保持しているのだと思います。このようなデータを利用することで特定の検索クエリに対し参照すべき partition を高速に判定できるようになるわけです。
```python
{
    "Martin": "Partition A",
    "John": "Partition B",
    "Kevin": "Partition C",
    ...
}
```

:::message
これはあくまでイメージです。実際の実装はもっと複雑でしょうし、後述する Search Method によっても保持するべきデータ構造は変わると思われます。「Dictみたな構造で列の値と Partition の対応関係を保持しておけば `O(1)` とかで参照すべき partition がわかるので partition pruning に使える」ということがキモです。
:::



## 用語

* Search Access Path
* Search Method
  * `FULL_TEXT` / `EQUALITY` / `SUBSTRING` / `GEO` の4つがある
  * それぞれに応じておそらく必要となるデータ構造が違う
  * 詳細は[こちら]( https://docs.snowflake.com/en/sql-reference/sql/alter-table#label-alter-table-searchoptimizationaction-add )をご覧ください



## Search Optimization が有効なクエリ

Search Optimization は追加でデータ構造を管理・維持しておくことで特定のクエリのパフォーマンスを向上する機能です。追加のコストも発生するため、「どのようなクエリのパフォーマンスを向上できるのか？」を理解しておくことは非常に重要です。


### 一般に言えること

以下のようなクエリのパフォーマンス向上に役立つ可能性があります。

* 元々数秒以上の時間がかかっていたクエリ
  * 実行時間が1秒未満のクエリは Search Optimization を有効化しても大幅にパフォーマンスが向上することはない
* where 句で指定している列のカーディナリティが高い（10万以上の異なる値を持つ）こと
* where 句でフィルターした結果、参照すべきパーティションが少ないこと
  * search optimization は partition pruning できる機会を増やすという仕組みなので、元々多数の partition を読み込まないといけないクエリの高速化はできない
* クラスタリングキー以外での絞り込みを含むクエリ
  * 


### データ型

* サポートされているデータ型
  * INTEGER / NUMERIC
  * DATE / TIME / TIMESTAMP
  * VARCHAR
  * BINARY
  * VARIANT / OBJECT / ARRAY
  * GEOGRAPHY
* 逆にサポートされていないデータ型
  * FLOAT や GEOMETRY


### テーブル種別

Search Optimization は普通のテーブルに対して有効化可能です。

一方外部テーブル・ビュー・マテリアライズドビューに対しては有効化できないので注意してください・



## 具体例

### ポイントルックアップ



### 部分文字列・正規表現での検索



### Join の最適化





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

* 法人番号についてはクラスタリングを行い、法人名の部分文字列検索を Search Optimization でサポートする



