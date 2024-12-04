---
title: "Snowflake ゼロコピークローン入門"
emoji: "🐏"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Snowflake", "DataEngineering", "SQL", ]
published: false
publication_name: finatext
---


## はじめに

こんにちは！ナウキャストのデータエンジニアのけびんです。


## ゼロコピークローンとは？


### Iceberg との類似性

:::message
ここで書くことは筆者独自の見解で、厳密に裏が取れているわけではありません。
:::




## 考慮事項



## ユースケース


### 複数環境作成

Prod 運用しているテーブルに何か変更を加えたい時に、まず Dev 環境を作りそこで作業したいというユースケースはあると思います。この際に Prod のテーブルをゼロコピークローンすることで Dev のテーブルをすぐに用意できます。

また CI でデータを利用したテストをしたいような場合にもゼロコピークローンは有用です。CI専用のデータベースやスキーマを本番環境のデータからゼロコピークローンして作成することで、短時間でCI専用の環境を用意することができます。

これらのように、複数環境を作成し安全にデータパイプラインの開発を進めたいケースはよくあります。新しい環境を作る際にゼロコピークローンを利用すると短時間でストレージコストも抑えながら複数の環境を用意できるわけです。


### WAP パターン

Write-Audit-Publish パターンと呼ばれるデータパイプラインの設計手法があります。
端的にいうと git ライクにデータを取り扱い、

* prod から dev ブランチを作る
* 新しいデータは dev ブランチに書き込む
* 新しいデータを含む dev ブランチのテーブルに対し品質チェックのテストを行う
* テストが通れば dev ブランチを prod ブランチに fast-forward マージする

というような手順を追うことで prod のテーブルの品質を担保するというものです。

以下の通り、 Iceberg の文脈などでよく取り上げられます。

![audit-branch](/images/articles/snowflake-zero-copy-clone/audit-branch.png =550x)
*https://iceberg.apache.org/docs/latest/branching/#audit-branch より*

https://speakerdeck.com/bering/apacheicebergthedefinitiveguidelun-du-hui-chapter14?slide=4


Snowflake のゼロコピークローンはまさに Git のブランチのような取り扱いが可能なので、このようなパターンの実装を低コストで行うことができます。

* prod テーブルから dev テーブルをゼロコピークローンで作成する
* dev テーブルに対しデータを書き込む
* 新しいデータを含む dev テーブルに対し品質チェックのテストを行う
* テストが通れば dev テーブルから prod テーブルをゼロコピークローンで上書きする

ナウキャストでもこれと同様にゼロコピークローンを取り扱うことで、コスパよくデータ品質を担保したデータパイプラインの作成を行っています。


### スナップショット






## References


https://docs.snowflake.com/en/sql-reference/sql/create-clone
https://docs.snowflake.com/en/user-guide/object-clone
https://docs.snowflake.com/en/user-guide/tables-storage-considerations#cloned-table-schema-and-database-storage

https://quickstarts.snowflake.com/guide/getting_started_with_snowflake/index.html#7
