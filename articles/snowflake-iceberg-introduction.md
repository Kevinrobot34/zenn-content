---
title: "Snowflake新機能： Iceberg Table と Polaris Catalog"
emoji: "❄️"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Snowflake", "Iceberg", "Data Engineering"]
published: false
---


## はじめに

こんにちは！ナウキャストのデータエンジニアのけびんです。
Data Cloud Summit 2024 にて Iceberg Table が GA となること、また Polaris Catalog が発表されました。大々的に発表されたので気になっているものの、詳細を知らない方も多いのではないでしょうか？

https://x.com/Kevinrobot34/status/1798428619687223776

自分もその一人だったので、本記事では改めて Apache Iceberg とは何かというところからまとめていきます。もし誤りなどあれば教えていただけますと幸いです。


## Table Format とは

Apache Iceberg とは大規模な分析データ向けの Open Table Format で、 Snowflake の Iceberg Table はこれを使用したテーブルということになります。Iceberg を深掘る前にそもそも Table Format とは何でしょうか？
Table Format とはファイルを管理・編成そして追跡し、テーブルを構成する方法のことです。関連する用語と並べて見ることで分かりやすくなると思います。

| Data Lakehouse の関連要素 |                    具体例                    |
| :-----------------------: | :------------------------------------------: |
|      Compute Engine       |    Spark / Presto / Hive / Snowflake など    |
|       Table Format        | **Iceberg** / Delta Table / Hive format など |
|        File Format        |       CSV / Avro / Parquet / ORC など        |
|      Object Storage       |      AWS S3 / GCS / Azure Storage など       |

Data Lakehouse としてなんらかの Object Storage に適当な File Format でデータを配置するだけではテーブルとしては有効に機能させられません。
物理的な個々のファイルをどのように活用することでテーブルとして理解できるか、ここの方法が大事であり、これが Table Format です。
物理的なデータの File Format と実際の構造化されたテーブルとの間に存在する抽象的なレイヤと考えることができます。

Table Format の具体例として Hive Format を見てみましょう。 Amazon Athena などで利用したことがある人も多いのではないでしょうか？
以下のサンプルのようにHive Format ではデータをディレクトリ構造で整理し、それぞれのパーティションに基づいて必要なファイルのみ読み込む Pruning ができるようになっているのがポイントです。
こうすることで、データ上に一種のインデックスを作成することができるわけです。
```
/hive/warehouse/sample_table
├── date=2024-01-01/
│   ├── category=CatA/
│   │   ├── file1
│   │   └── file2
│   └── category=CatB/
│       └── file3
├── date=2024-01-02/
│   ├── category=CatA/
│   │   └── file4
│   └── category=CatB/
│       ├── file5
│       └── file6
...
```
Iceberg もこのような Table Format の一つです。


## 既存の Table Format の問題点

Hive Format などの既存の Table Format にはいくつか問題点がありました。

* Schema Evolution / Partition Evolution をサポートしていない
  * 
* ACID 特性がない
* Time Travel できない
* ...
* ( https://iceberg.apache.org/spec/#goals の話)

これらを解消すべくここ最近に作られた Open Table Format の一つが Iceberg というわけです。


## Iceberg の詳細

Iceberg はどのようにデータを編成するのかをみてみましょう。

![iceberg-metadata](/images/articles/snowflake-iceberg-introduction/iceberg-metadata.png)
*https://iceberg.apache.org/spec/#overview より*

a persistent tree structure

* architecture layer
  * **Iceberg Catalog**
* metadata layer
  * metadata file / snapshot
  * manifest list
  * manifest file
* data layer


## Catalog について


## Iceberg Table について


![tables-iceberg-snowflake-as-catalog](/images/articles/snowflake-iceberg-introduction/tables-iceberg-snowflake-as-catalog.png)
*https://docs.snowflake.com/ja/user-guide/tables-iceberg#use-snowflake-as-the-iceberg-catalog より*


![tables-iceberg-external-catalog](/images/articles/snowflake-iceberg-introduction/tables-iceberg-external-catalog.png)
*https://docs.snowflake.com/ja/user-guide/tables-iceberg#use-a-catalog-integration より*



## Catalog の問題点と Polaris Catalog


* Instead of moving and copying data for different engines and catalogs, you can interoperate many engines on a single copy of data from one place.
* You can host it in Snowflake managed infrastructure or your infrastructure of choice.

https://www.snowflake.com/blog/introducing-polaris-catalog/?lang=ja

## 具体例

### Iceberg Table を作ってみる



### Time-Travel っぽい実例


## まとめ



## References

### Snowflake blog

https://www.snowflake.com/blog/unifying-iceberg-tables/

https://www.snowflake.com/blog/build-open-data-lakehouse-iceberg-tables/

https://www.snowflake.com/blog/introducing-polaris-catalog/?lang=ja

### その他

https://medium.com/expedia-group-tech/a-short-introduction-to-apache-iceberg-d34f628b6799
https://medium.com/geekculture/open-table-formats-delta-iceberg-hudi-732f682ec0bb

