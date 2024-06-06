
## メモ

* keyword
  * iceberg
  * table format
    * https://medium.com/geekculture/open-table-formats-delta-iceberg-hudi-732f682ec0bb
  * catalog
  * lakehouse
  * schema evolution
  * hidden partitioning
  * time travel
  * atomicity
  * 並行制御・排他ロック
    * https://agiledata-ja.github.io/ImplementingConcurrencyControl
    * https://zenn.dev/airiswim/articles/ebe313fb39a4c9
  * 結果生合成と強い生合成
  * a persistent tree structure
    * https://medium.com/@pushp.agarwal/use-of-persistent-tree-in-versioning-and-transactions-d4c60c555747
* plot
  * Iceberg とは
  * Iceberg の特性
    * ACID（原子性、一貫性、分離性、耐久性）トランザクション / ACID (atomicity, consistency, isolation, durability) transactions
      * https://en.wikipedia.org/wiki/ACID
      * Atomicity 原子性
        * Transactions are often composed of multiple statements. Atomicity guarantees that each transaction is treated as a single "unit", which either succeeds completely or fails completely: if any of the statements constituting a transaction fails to complete, the entire transaction fails and the database is left unchanged.
      * Consistency 一貫性
        * トランザクションの前後で定義したルールが満たされていることを保証する
      * Isolation 分離性
        * トランザクションが並列に同時に実行されても、逐次処理した際と同じ状態になることを保証すること
        * 並行処理制御におけるゴールはIsolation
          * 楽観的並行性制御（楽観的ロック） / 悲観的並行性制御（悲観的ロック）
          * https://zenn.dev/airiswim/articles/ebe313fb39a4c9
      * Durability 耐久性
        * トランザクションがコミットされたら永続化されるよ的な
    * スキーマの進化 / Schema evolution
    * 隠しパーティション / Hidden partitioning
    * テーブルスナップショット / Table snapshots
  * Native Iceberg table と External Iceberg Table
  * 
* Reference
  * Snowflake
    * quickstart
      * https://quickstarts.snowflake.com/guide/data_lake_using_apache_iceberg_with_snowflake_and_aws_glue/index.html#0
      * https://quickstarts.snowflake.com/guide/getting_started_iceberg_tables/index.html#0
    * document
      * https://docs.snowflake.com/ja/user-guide/tables-iceberg
    * blog
      * https://www.snowflake.com/blog/unifying-iceberg-tables/
      * https://www.snowflake.com/blog/build-open-data-lakehouse-iceberg-tables/
      * https://www.snowflake.com/blog/introducing-polaris-catalog/?lang=ja
      * 
 *  Iceberg
    * https://iceberg.apache.org/spec/
    * https://iceberg.apache.org/docs/latest/
    * https://iceberg.apache.org/concepts/catalog/
    * https://github.com/apache/iceberg
  * others
    * https://medium.com/expedia-group-tech/a-short-introduction-to-apache-iceberg-d34f628b6799
    * https://medium.com/geekculture/open-table-formats-delta-iceberg-hudi-732f682ec0bb
    * https://zenn.dev/shigeru_oda/articles/05dbf435200b97e87ee4
    * https://x.com/_Bassari/status/1798006269523243336
    * https://www.snowflake.com/webinars/thought-leadership/ask-the-experts-polaris-catalog-2024-07-23/
    * 

