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


https://speakerdeck.com/tomtanaka/apache-iceberg-the-definitive-guide-lun-du-hui-di-6-zhang-apache-spark

### 7: Dremio’s SQL Query Engine

https://speakerdeck.com/tanisuhi/iceberg-definitive-guidelun-du-hui-chapter7-and-8

### 8: AWS Glue

https://speakerdeck.com/tanisuhi/iceberg-definitive-guidelun-du-hui-chapter7-and-8

### 9: Apache Flink

https://speakerdeck.com/tanisuhi/iceberg-definitive-guidelun-du-hui-chapter9




## Part3: Apache Iceberg in Practice

Part1 で

### 10: Apache Iceberg in Production

10章では Iceberg を本番運用していく中で大事となる様々な Tips が紹介されています。具体的には

* Metadata Tables
* Branching and Tagging
* Multi-table Transactions
* Rollback

などです。これらを reactive または proactive に適切に利用していくことで、安全かつ信頼性の高い Iceberg table の運用が可能となります。


https://speakerdeck.com/riu/apache-iceberg-the-definitive-guide-lun-du-hui-10zhang-apache-iceberg-in-production-qian-ban-bu-fen


https://zenn.dev/dataheroes/articles/iceberg-the-definitive-guide-ch10


### 11: Streaming with Apache Iceberg

11章では Iceberg を Streaming data に対して利用する際の話がまとめられています。
Streaming data はデータ量が多く、スキーマも変化していく可能性も高いデータですが、 Iceberg はこれらを取り扱えるように設計されているため、相性は良いわけです。
Spark や Flink 、 Kafka Connect を Streaming data に利用する方法、またその際の注意点などがまとめられています。

https://zenn.dev/dataheroes/articles/iceberg-the-definitive-guide-ch11


### 12: Governance and Security

12章では Iceberg のガバナンスとセキュリティについてまとめられています。
Iceberg 自身には実はセキュリティの要件は直接は定義されていません。 Iceberg を本格的に運用していくと、 Iceberg のデータファイルやメタデータファイル、そしてカタログがデータ・メタデータの中心になっていくため、ここのセキュリティを強固にしておくことは非常に重要になります。

この章ではファイルレベル・セマンティックレイヤーレベル・カタログレベルそれぞれで考えるべきガバナンスやセキュリティについてまとめられています。

https://zenn.dev/dataheroes/articles/iceberg-the-definitive-guide-ch12


### 13: Migrating to Apache Iceberg

13章では Iceberg へマイグレーションする方法やその際の注意点についてまとめられています。
新規のデータパイプラインやデータ基盤で Iceberg を利用する場合にはこれまでの章の情報でも十分かもしれませんが、
既存のデータパイプラインやデータ基盤を Iceberg に移行するとなると追加で考えるべきことは増えます。

この章ではインプレース移行やシャドウ移行といった以降の方針や具体的な移行方法について述べられています。

https://zenn.dev/dataheroes/articles/snowflake-iceberg-migrating


### 14: Real-World Use Cases of Apache Iceberg

14章は最終章ということで、実世界での Iceberg の具体的なユースケースが紹介されています。
* Write-Audit-Publish (WAP) によるデータ品質管理
* Dremio SQL Query Engine を活用した BI レポートの最適化
* Iceberg による Change Data Capture (CDC)

といった事例が具体的に紹介されています。

https://speakerdeck.com/bering/apacheicebergthedefinitiveguidelun-du-hui-chapter14