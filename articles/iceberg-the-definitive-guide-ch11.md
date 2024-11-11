---
title: "Apache Iceberg: The Definitive Guid 11章 Streaming with Apache Iceberg"
emoji: "🧊"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Iceberg", "DataEngineering", "OTF", "Snowflake"]
published: false
publication_name: dataheroes
---


## はじめに

こんにちは！ナウキャストでデータエンジニアをしているけびんです。

SnowVillage で行っている [Apache Iceberg: The Definitive Guid]( https://www.oreilly.com/library/view/apache-iceberg-the/9781098148614/ ) 輪読会の11章の資料です。11章は "Streaming with Apache Iceberg" というタイトルで Iceberg でストリーミング処理する話が書かれていますが、その内容をまとめたり、調べて見つけた関連した内容も適宜記載したりしています。

:::message
この形式が私の感想・コメントです
:::


### 11章の introduction

ストリーミングデータとはさまざまなソースから継続的に生成され処理されるデータのことを指します。ログファイルやセンサーデータ、ソーシャルメディアのフィードなど、様々なデータソースから日々データは生成されています。これらのデータが小さいサイズに分割され送信され、リアルタイムで処理されるわけです。

ストリーミングデータは現在さまざまな場所で利用されていますが、これらを Apache Iceberg table として取り込むメリットは以下の通りです。

* Scalability and performance
* Schema evolution
* Reliability
* Time travel


## Streaming into Iceberg with Spark






Streaming from Iceberg with Spark
Streaming with Flink
Streaming into Iceberg with Flink
Example of Streaming into Iceberg with Flink
Streaming with Kafka Connect
The Iceberg Kafka Sink
Streaming with AWS
Conclusion
