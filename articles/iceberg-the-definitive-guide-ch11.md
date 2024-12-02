---
title: "Apache Iceberg: The Definitive Guide 11章 Streaming with Apache Iceberg"
emoji: "🧊"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Iceberg", "DataEngineering", "OTF", "Snowflake"]
published: true
publication_name: dataheroes
---


## はじめに

こんにちは！ナウキャストでデータエンジニアをしているけびんです。

SnowVillage で行っている [Apache Iceberg: The Definitive Guid]( https://www.oreilly.com/library/view/apache-iceberg-the/9781098148614/ ) 輪読会の11章の資料です。11章は "Streaming with Apache Iceberg" というタイトルで Iceberg でストリーミング処理する話が書かれていますが、その内容をまとめたり、調べて見つけた関連した内容も適宜記載したりしています。

:::message
この形式が私の感想・コメントです
:::


## 11章の introduction

ストリーミングデータとはさまざまなソースから継続的に生成され処理されるデータのことを指します。ログファイルやセンサーデータ、ソーシャルメディアのフィードなど、様々なデータソースから日々データは生成されています。これらのデータが小さいサイズに分割され送信され、リアルタイムで処理されるわけです。

ストリーミングデータは現在さまざまな場所で利用されていますが、これらを Apache Iceberg table として取り込むメリットは以下の通りです。

* Scalability and performance
  * Iceberg は巨大なデータセットの保存・情報抽出を効率的に行えるように設計されている
  * ファイル管理のプロシージャも定義されており、パフォーマンスを維持するための仕組みも作りやすい
* Schema evolution
  * 時々刻々とデータが送られてくるストリーミングデータではスキーマが変化する可能性も高い
  * Iceberg であれば Schema Evolution をサポートしており、変化に追従しながらデータを扱うことが可能
* Reliability
  * Iceberg の Snapshot や Transaction の仕組みが一貫して信頼性の高くデータを取り扱うのに適している
* Time travel
  * 時々刻々と溜まるデータに対してタイムトラベルでクエリできると便利

ここから先では具体的に各種ツールでストリーミングデータを取り扱う方法を見ていきます。


## Streaming into Iceberg with Spark

Spark の特徴の一つはストリーミングデータを処理することが可能であることです。
Spark Streaming を利用することでスケーラブルで、スループットも信頼性も高いストリーム処理を行うことが可能になります。

マイクロバッチのアプローチを取る Spark Streaming の具体的な特徴は以下の通りです

* Fault tolerance
  * ビルトインのリカバリー処理のおかげで耐障害性が高い
  * 各マイクロバッチの処理を監視し、失敗したらそのマイクロバッチをリトライすれば良いので、耐障害性が高くなります
* Integration
  * Spark Streaming は Spark MLlib や Spark SQL など Spark の他のコンポーネントとシームレスに連携することができる
  * 他のバッチ処理の技術をリアルタイム処理に活かしやすい
* Real-time processing
  * Spark streaming はストリーミングデータをマイクロバッチに分割し、これを Spark engine で処理するという構成をとっており、これによりストリーミングデータを次々と処理することが可能
* Window operations
  * ウィンドウに対する様々な計算がサポートされていて、ストリーミングデータの分析に便利
* High throughput
  * ハイボリュームなストリーミングデータを処理する小tが可能
* Multiple data sources
  * Kafka / Flume / Kinesis など様々なデータソースからストリーミングデータを取り込むことが可能

このように全体としてマイクロバッチ処理のアプローチは応答性、処理可能なデータ量、リソース効率のバランスが取れており、ビッグデータやストリーミング処理の領域で好まれる選択肢となっています。


この後の内容は以下にも書かれています。
https://iceberg.apache.org/docs/1.6.1/spark-structured-streaming/


### Streaming into Iceberg with Spark


Spark の `DataStreamWriter` を利用することで Iceberg テーブルにストリーミング方式で書き込むことができます。
その際には方法は以下の二つがあります。
* append   : 各マイクロバッチの行をテーブルに追加する
* complete : 各マイクロバッチでテーブルの内容を置き換える


パーティション化されているテーブルに関しては少し注意が必要です。パーティション化されたテーブルはパーティションごとにデータをソートしておく必要がありますが、バッチごとにソートしているとレイテンシーが増えてしまいます。これをバイパスするために fanout writer オプションというのがあります。これを有効化すると、書き込みの際にソートの必要がなくなりパフォーマンスを上げることができます。


またストリーミングデータはすぐに新しいテーブルのバージョンを作成してしまい、細かいメタデータファイルやデータファイルが生まれてしまいます。効率的なメタデータ管理、そしてクエリパフォーマンスのためには以下のような対応が重要です
* コミットのレートを調整する
* 古いスナップショットは廃棄する
* データファイルの compaction を行う
* マニフェストの rewrite を行う

この辺りは4章で詳細を取り扱っているのでぜひご覧ください。

https://speakerdeck.com/kevinrobot34/apache-iceberg-the-definitive-guide-ch4

https://speakerdeck.com/rshimajiri/apache-iceberg-the-definitive-guide-lun-du-hui-4zhang-optimizing-the-performance-of-iceberg-tables-hou-ban


kafka から読み込み、 iceberg に書き込むサンプルコードは以下の通りです。

```
val df = spark.readStream
    .format("kafka")
    .option("kafka.bootstrap.servers", "localhost:9092")
    .option("subscribe", "financialData")
    .load()

val financialData = df.selectExpr("CAST(value AS STRING)").as[String]
    .map(Stock.from_json(_))

case class Stock(timestamp: Long, symbol: String, price: Double, volume: Long)
object Stock {
  def from_json(jsonString: String): Stock = {
    val parser = new ObjectMapper()
    parser.registerModule(DefaultScalaModule)
    parser.readValue(jsonString, classOf[Stock])
  }
}

val tableIdentifier = "s3://someBucket/financial_data/stock"
financialData.writeStream
    .format("iceberg")
    .outputMode("append")
    .trigger(Trigger.ProcessingTime(1, TimeUnit.MINUTES))
    .option("path", tableIdentifier)
    .option("checkpointLocation", "/tmp/checkpoints")
    .start()
```


### Streaming from Iceberg with Spark


DataSource V2 API を利用することで以下のように Iceberg テーブルからストリーミングでデータを読み込むことができます。

```
val df = spark.readStream
    .format("iceberg")
    .option("stream-from-timestamp", Long.toString(streamStartTimestamp))
    .load(tableName)
```

この際に注意点としては以下の通りです。
* append snapshot の Iceberg テーブルからの読み込みのみがサポートされている
* 下流のアプリケーションを Iceberg からのリアルタイムストリーミングデータを取り扱えるように設計しておくこと


:::message
append snapshot は snapshot の summary の operation は append になっている snapshot ということだろうか？明示的に定義しているところを見つけられなかった。関連リンク
* https://iceberg.apache.org/spec/#snapshots
* https://github.com/developer-advocacy-dremio/definitive-guide-to-apache-iceberg/blob/main/Resources/Chapter_2/metadata-file.json
:::


## Streaming with Flink

Apache Flink はオープンソースの統合ストリームバッチ処理エンジンです。大量のデータを高速で処理するように設計されており、リアルタイムデータ処理・分析に役立ちます。

Apache Flink の特徴は以下の通りです
* event time semantics のサポート
* exactly-once semantics のサポート
* backpressure control


* Event time processing
  * Event が起きた順番に処理を行うことができるという性質
  * Event の順序が重要である金融のトランザクションなどで非常に重要になります
  * 詳細は以下のスライドが詳しいです
    https://speakerdeck.com/sharonx/timing-is-everything-understanding-event-time-processing-in-flink-sql-current-24
* Fault tolerance
  * チェックポイント・セーブポイント機構により、耐障害性が高い
  * データを処理中にロストしたりしない
* Backpressure handling
  * 消費可能な量より大量のデータが生成されてしまった際にも対応が可能
  * これにより大量なデータが流れてきてもシステムは安定したままです
* High throughput and low latency
  * 大量のデータを低いレイテンシーで捌けるように設計されており、リアルタイムデータ分析に向いている
* Windowing and complex event processing
  * さまざまなウィンドウ関数でデータの処理が可能
* Integration
  * Kafka や HDFS そして Cassandra などさまざまなデータソースやsinkと連携が可能
* Exactly-once semantics
  * どのレコードも厳密に一度だけ取り込まれることが保証されており、正確なデータ処理が可能に


このように Apache Flink は高速で正確でスケーラブルなストリーミングデータ処理が可能で、上記のような各種特徴からリアルタイムデータ処理の強力な選択肢となっています。


## Streaming with Kafka Connect


Apache Kafka Connect は Kafka platform のオープンソースのコンポーネントであり、Kafkaとの間でデータを移動するためのスケーラブルで信頼性の高い方法を提供します。

以下のような特性があります。

* High throughput
* Fault tolerance
* Real-time integration
* Durability
* Wide integration
* Exactly-once semantics
* Connector framework


Apache Kafka Connect はKafka エコシステム内のロバストな統合ツールで、シームレスにリアルタイムデータ連携機能を提供します。


