---
title: "ハイパフォーマンス dbt incremental model"
emoji: "🤖"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["dbt", "DataEngineering", "SQL", "Snowflake"]
published: false
publication_name: finatext
---

この記事は、[dbt Advent Calendar 2024]( https://qiita.com/advent-calendar/2024/dbt )の8日目の記事です。

## はじめに

こんにちは！ナウキャストのデータエンジニアのけびんです。


## dbt incremental model とは

dbt を利用すると ELT パイプラインを宣言的に作成・管理することができます。 `hoge.sql` というファイルに select 文を書いておくと、その内容で `hoge` というテーブルを作るように CTAS でラップして実行してくれる、という訳です。これはつまりテーブル全体を毎回洗い替えする構成になってしまいますが、データ量が大きいテーブルに対して毎回洗い替えするわけにはいきません。

そこで dbt における materialization の一種である incremental model を利用すると、洗い替えではなくいわゆる差分更新を可能にします。  incremental model の詳細については以下のドキュメントなどをご覧ください。

https://docs.getdbt.com/docs/build/incremental-models-overview

https://docs.getdbt.com/best-practices/materializations/1-guide-overview

incremental model は非常に便利なのですが、 table の materialization に比べ複雑で難しいです。本ブログでは incremental model をパフォーマンスよく実装するためのプラクティスや考え方を紹介します。


## 適切な Partitioning

incremental model はいわゆる差分更新をするための仕組みです。 incremental model を利用するデータに対して適切に「差分」を定義し、それをフィルタリングできるようにテーブルを構成しておいてあげることが大事です。
例えば、ある巨大なテーブルに row_id という UUID のユニークな列があったとして、これをkeyにして差分更新しようとすると、差分を特定するために毎回テーブルのフルスキャンをする必要があったりします。
なんらかの日付のような適切なパーティションを探してそれを利用して incremental model を使ってみるのが大事です。
peiさんの以下の記事が非常にわかりやすくこの辺りの話を書いてくれているので是非読んでみてください。

https://zenn.dev/pei0804/articles/data-partitioning-in-dbt


## コンパイル後のクエリと向き合う

dbt incremental model は差分更新をするために、複数のクエリを実行することがあります。例えば incremental strategy として delete+insert を選択すると
CTAS: 差分データに対応する一時テーブルを作成
DELETE: 差分テーブルと元テーブルを突き合わせ delete を実行
INSERT: 差分テーブルを元テーブルに insert
と３つのクエリが実行されます。
この場合、１つ目のCTASは比較的モデルファイルのクエリがそのままな形ですが、２つ目のDELETEと３つ目のINSERTはモデルファイルのconfigからよしなに生成されています。
これらのクエリが予期せぬ高コストなものになっていることがあったりします。

これらを踏まえ、dbt incremental model はどのようなクエリを生成しているのかを確認し、またそのクエリの効率が悪くないか実際に確かめることが非常に重要になってきます。

自分は以前、dbt-snowflakeで delete+insert の incremental model を作成した際に、 ２つ目のDELETEのクエリで exploding join となってしまっておりものすごく時間がかかるモデルになってしまっていた、という経験をしました。詳細は以下をご覧ください。
https://techblog.finatext.com/dbt-snowflake-incremental-exploding-joins-7ca8a6b484ca

:::message
「コンパイル後のクエリと向き合う」という姿勢は incremental model 以外でも、dbtを利用している時に全般的に持っておくことが大事です。例えば dbt test は便利な機能ですが、実際に実行されているクエリがどのようなものか見たことあるでしょうか？設定によっては意味のないテストになっていたり、非効率なものになっていたりすることも見かけます。是非チェックしてみましょう。
:::


### コンパイル後のクエリの生成方法

コンパイル後の実際に実行しているクエリと向き合うことが重要なのは分かったとして、どのようにコンパイル後のクエリを確認するのかという話が出てきます。

一番シンプルなのはクエリの実行履歴を確認し、 run や build が実行したクエリを DWH 側で確認する、という方法です。既に incremental model を運用している場合などではまずこれをやって見るのが良いでしょう。例えば Snowflake であれば Query History から Query Profile を確認し、非効率になっているクエリがないかを確認することが大事です。
また Snowflake であれば SELECT などのツールを利用すると dbt のモデルごとに Query History の情報をまとめて表示してくれるため非常に便利です。


![select-sample](/images/articles/dbt-high-performance-incremental-model/select-sample.png =500x)

https://findy-tools.io/products/select/385/333


## incremental_predicates
https://docs.getdbt.com/docs/build/incremental-strategy#about-incremental_predicates



## References
https://docs.getdbt.com/best-practices/materializations/1-guide-overview


