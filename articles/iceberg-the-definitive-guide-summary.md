---
title: "Apache Iceberg: The Definitive Guid 輪読会まとめ"
emoji: "🐻‍❄️"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Snowflake", "Iceberg", "DataEngineering"]
published: false
publication_name: dataheroes
---

## はじめに

こんにちは！ナウキャストのデータエンジニアのけびんです。

今年の6月に Iceberg Table が Snowflake の機能として GA したのは記憶に新しいかと思います。
自分もこの時から Iceberg に興味を持ちブログを書いたりしました。
https://zenn.dev/dataheroes/articles/snowflake-iceberg-introduction

そんな中、ちょうど良いタイミングで [Apache Iceberg: The Definitive Guid]( https://www.oreilly.com/library/view/apache-iceberg-the/9781098148614/ ) が2024年5月に出版されており、 SnowVillage の有志の方たちと輪読会という形で読み進めておりました。

11月末に無事に全体を読み終えましたので、今回は各章について簡単に紹介していきたいと思います。

## Part1: Fundamentals of Apache Iceberg

Part1 は Apache Iceberg の基礎、ということで、 Iceberg が生まれてきた歴史や、そのアーキテクチャや仕組み、カタログなどについて解説されています。このパートを読めば Iceberg がどのような技術なのか、全体像が掴めるはずです。

### 1: Introduction to Apache Iceberg

１章では入門としてまずデータエンジニアリングの歴史について簡単に振り返っています。これらを知ることでなぜ Iceberg のような Open Table Format が必要になってきたのか、が理解できるようになるでしょう。またデータエンジニアであれば Iceberg などの OTF にいたるまでの歴史を知ることは興味深くまた勉強になるかと思います。

歴史を見た後には Iceberg が持つさまざまな特性や機能についても説明があります。 Iceberg の概要を知りたいだけであればこの１章を読めば済むことも多いかなと思います。


https://speakerdeck.com/efexp/iceberg-the-definitive-guide-lun-du-hui-di-hui

https://speakerdeck.com/kevinrobot34/apache-iceberg-the-definitive-guide-ch1


### 2: The Architecture of Apache Iceberg

2章では Iceberg のアーキテクチャを深掘ります。データファイルとメタデータファイルの階層構造の説明や、それがどのように Iceberg の各種特性や機能に役立っているのかが具体的に理解できるようになるはずです。

https://speakerdeck.com/bering/apache-iceberg-the-definitive-guide-lun-du-hui-2zhang-the-architecture-of-apache-iceberg


### 3: Lifecycle of Write and Read Queries

3章ではクエリのライフサイクルを見ていきます。これにより、各種データファイルやメタデータファイル、そしてカタログがどのように連携して実際にクエリを実行するのかが、より具体的にわかるようになるはずです。

具体的なサンプルのコードも多いため、実際に手を動かしながら Iceberg の理解を手助けしてくれる章となっています。

https://speakerdeck.com/riu/apache-iceberg-the-definitive-guide-lun-du-hui-3zhang-lifecycle-of-write-and-read-queries


### 4: Optimizing the Performance of Iceberg Tables

4章では Iceberg の最適化に関してさまざまなトピックが解説されています。具体的には
* Compaction
* Clustering (Sorting / Z-ordering)
* Partitioning
* Row-level update (CoW vs. MoR)
* Metrics collection

といったトピックについて解説されています。Iceberg も銀の弾丸ではなく、アーキテクチャや仕組みを踏まえ適切に設定することが大事になるとよく分かるかと思います。

https://speakerdeck.com/kevinrobot34/apache-iceberg-the-definitive-guide-ch4

https://speakerdeck.com/rshimajiri/apache-iceberg-the-definitive-guide-lun-du-hui-4zhang-optimizing-the-performance-of-iceberg-tables-hou-ban


### 5: Iceberg Catalogs

5章では Iceberg Catalog について深掘ります。
Iceberg において Catalog はテーブルとそれが指し示す最新のメタデータファイルの情報を管理し、その Atomic な更新をサポートしていれば良い、というシンプルなものです。
しかし実際にはさまざまなカタログの実装があり、それぞれ独自の機能があったりします。この章ではこの辺りの Catalog にまつわる話題を確認することが可能です。

https://zenn.dev/musyu/articles/5d9ee475f5f51a



## Part2: Hands-on with Apache Iceberg

### 6: Apache Spark


### 7: Dremio’s SQL Query Engine


### 8: AWS Glue


### 9: Apache Flink


