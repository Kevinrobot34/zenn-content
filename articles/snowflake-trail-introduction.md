---
title: "新機能 Snowflake Trail の紹介"
emoji: "❄️"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Snowflake", "Observability", "OpenTelemetry"]
published: false
---

## はじめに

こんにちは！ナウキャストのデータエンジニアのけびんです。
Data Cloud Summit 2024 にて Snowflake Trail という Observability に関する機能群が発表されました！

https://www.snowflake.com/en/data-cloud/snowflake-trail/

## Observability とは

ソフトウェアエンジニアリングにおいて Observability とは可観測性と訳され、システムやアプリケーションの出力・ログ・パフォーマンス指標などを調べることにより、システムがおかれている状態を監視・測定・理解できる能力のことです。

Observability が高いシステムはデバッグやメンテナンスが容易になり、信頼性・安定性が高まります。Snowflake は AI Data Cloud として様々なアプリ、パイプライン、そしてデータが集まるプラットフォームとなっていますが、今後更なる活用を広げるためにも Observability の重要性がますます高まっていると言えると思います。


## Snowflake Trail

Snowflake Trail は Snowflake の様々な構成要素の可観測性を高めるための機能群です。
Event Table / Alert & Notification をその基礎とし、 Pipeline / App / Data の３つそれぞれの Observability を高めるための機能が随時開発されていきます。

* Event Table
  * https://docs.snowflake.com/ja/developer-guide/logging-tracing/event-table-operations
* Alert & Notification
* Pipeline Observability
* App Observability
* Data Observability
* 