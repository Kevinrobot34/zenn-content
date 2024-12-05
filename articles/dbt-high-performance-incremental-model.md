---
title: "ハイパフォーマンス dbt incremental model"
emoji: "🤖"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["dbt", "DataEngineering", "SQL"]
published: false
publication_name: finatext
---

この記事は、[dbt Advent Calendar 2024]( https://qiita.com/advent-calendar/2024/dbt )の8日目の記事です。

## はじめに

こんにちは！ナウキャストのデータエンジニアのけびんです。


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


## incremental_predicates
https://docs.getdbt.com/docs/build/incremental-strategy#about-incremental_predicates



## References
https://docs.getdbt.com/best-practices/materializations/1-guide-overview


