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

Snowflake は様々な便利な機能がありますが、その一つに Search Optimization があります。 Search Optimization は大規模なテーブルに対する選択的な検索などのクエリのパフォーマンスを向上させるための仕組みです。

https://docs.snowflake.com/ja/user-guide/search-optimization-service


本ブログでは Search Optimization に関する概要を説明し、利用時の注意点や具体的なユースケースを紹介していきます。



## 仕組み

クエリのパフォーマンスを向上させるためには余計なマイクロパーティションを読み込まないことが重要ですが、 Search Optimization では "Search Access Path" と呼ばれる追加のデータ構造を維持・管理し、それを利用することで、選択的な検索等のクエリの際に pruning を支援します。

![snowflake-arch](/images/articles/snowflake-search-optimization/how-it-works.png =500x)
*https://quickstarts.snowflake.com/guide/getting_started_with_search_optimization/index.html#0 より*


イメージとしては、 Python の Dict のような構造をで特定の列の値と Partition の対応関係を保持できるようにし、これを利用することで特定の検索クエリに対し参照すべき partition を高速に判定できるようにする、というようなものです。
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


このように追加のデータ構造を維持することになるため、その分のストレージコストとコンピュートのコストがかかります。このコストと、Search Optimization 導入によるパフォーマンスの向上を


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

Search Optimization は `ALTER TABLE ... ADD SEARCH OPTIMIZATION` でテーブルに対し有効化しますが、設定時には以下の権限が必要です。

* テーブルに対する **OWNERSHIP** 権限
* テーブルを含む**スキーマ**に対する **ADD SEARCH OPTIMIZATION** 権限
  * おそらくスキーマの中で新たなデータ構造として search access path を作るため、 ADD SEARCH OPTIMIZATION 権限はスキーマの権限の一つなのだと思います


Search Optimization による最適化を行うためには特に追加の権限は必要ありません。 Search Optimization が有効化されたテーブルに対する SELECT 権限があれば、自動的に利用可能となっています。コンパイル時に必要と判断されれば Search Optimization が勝手に使われるだけという形です。



## コスト

Search Optimization を有効化するとコストが発生します。このコスト増加と Search Optimization によるコスト削減のトレードオフがあるので、 Search Optimization にかかるコストを理解し、適切に監視しながら利用することが大切になります。


### ストレージコスト

テーブルとは別に Search Access Path を追加で維持・管理することになるのでこの分のストレージコストが発生します。
極端な例で考えると、全ての列がユニークなテーブルに対して Search Optimization を有効化すると、 Search Access Path は下のテーブルと同等のサイズになる可能性があります。しかしドキュメントによると通常、サイズは元のテーブルの約1/4となるそうです。



### コンピューティングコスト

Search Optimization を有効化すると、 Search Access Path が構築・維持されますが、その際にコンピューティングコストが発生します。 Search Optimization はサーバレス機能の一つなため、この際に使われるのは Snowflake managed なコンピュートリソースです。


自動クラスタリング は、検索最適化を使用してテーブル内のクエリの遅延を改善できますが、検索最適化のメンテナンスコストをさらに増加させる可能性があります。テーブルのチャーンレートが高い場合は、自動クラスタリングを有効にしてテーブルの検索最適化を構成すると、テーブルが検索最適化用に構成されている場合よりもメンテナンスコストが高くなる可能性があります。



大きなテーブル（テラバイト（TB）以上のデータを含むテーブル）に検索最適化を追加すると、短期間にクレジット消費量が即座に増加する可能性があります。


テーブルに検索最適化を追加すると、メンテナンスサービスは、すぐにバックグラウンドでテーブルの検索アクセスパスの構築を開始します。テーブルが大きい場合、メンテナンスサービスがこの作業を大規模に並列化する可能性があり、その結果、短期間にコストが増加する可能性があります。


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
  * 自動クラスタリング は、検索最適化を使用してテーブル内のクエリの遅延を改善できますが、検索最適化のメンテナンスコストをさらに増加させる可能性があります。テーブルのチャーンレートが高い場合は、自動クラスタリングを有効にしてテーブルの検索最適化を構成すると、テーブルが検索最適化用に構成されている場合よりもメンテナンスコストが高くなる可能性があります。



## ナウキャストで検討しているユースケース

### 法人番号と法人名

* 法人番号についてはクラスタリングを行い、法人名の部分文字列検索を Search Optimization でサポートする




## References

* https://docs.snowflake.com/en/sql-reference/sql/alter-table#label-alter-table-searchoptimizationaction-add
* 