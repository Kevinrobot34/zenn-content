---
title: "dbt incremental model で大規模データを取り扱うプラクティス"
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

incremental model は非常に便利なのですが、 table の materialization に比べ複雑で難しいです。本ブログでは incremental model をパフォーマンスよく実装するためのプラクティスや考え方を紹介します。


## 適切な Partitioning

incremental model のイメージとしては、以下の図のように、ソーステーブルから差分に該当するレコードだけ抽出してきて、それを既存のモデルに追加・更新します。incremental model のパフォーマンスを高めるには差分に該当するレコードをソーステーブルから抽出する操作をいかに効率化するかがまず大事になります。

![incremental-diagram](/images/articles/dbt-high-performance-incremental-model/incremental-diagram.png =500x)
*https://docs.getdbt.com/best-practices/materializations/4-incremental-models より*

具体例で考えてみましょう。例えば、ある巨大なテーブルに `row_id` という UUID の列があったとして、これをkeyにして差分更新するように設定したとします。この `row_id` 列はテーブル全体でユニークなため、差分を特定するために毎回テーブルのフルスキャンをする必要になるわけです。これでは差分を抽出する操作が非効率で、テーブルが大きくなるにつれかかる時間も増えてしまいます。

実際には、日付など差分を特定するために利用できる適切な列があることが多いでしょう。この列を利用してソーステーブルを partitioning しておけば、ソーステーブルから差分を抽出する際に partition pruning が利用できるようになり、フルスキャンを回避できます。このように **incremental model の実装をするというのはパーティションの設計をする、というのとほぼ同義**なのです。

peiさんの以下の記事でこの辺りの話が非常にわかりやすく解説されているので是非読んでみてください。
https://zenn.dev/pei0804/articles/data-partitioning-in-dbt


## コンパイル後のクエリと向き合う

dbt incremental model は差分更新をするために、複数のクエリを実行することがあります。例えば incremental strategy として delete+insert を選択すると
* CTAS: 差分データに対応する一時テーブルを作成
* DELETE: 差分テーブルと元テーブルを突き合わせ delete を実行
* INSERT: 差分テーブルを元テーブルに insert

と３つのクエリが実行されます。この場合、１つ目のCTASは比較的モデルファイルのクエリがそのままな形ですが、２つ目のDELETEと３つ目のINSERTはモデルファイルのconfigからよしなに生成されます。これらのクエリが予期せぬ高コストなものになっていることがあるわけです。

自分は以前、 dbt-snowflake で delete+insert の incremental model を作成した際に、 ２つ目のDELETEのクエリで exploding join となってしまっておりものすごく時間がかかるモデルになってしまっていた、という経験をしました。この件については別の記事で解説しておりますので、詳細は以下をご覧ください。
https://techblog.finatext.com/dbt-snowflake-incremental-exploding-joins-7ca8a6b484ca

このように、dbt incremental model がどのようなクエリを生成しているのかを確認し、またそのクエリの効率が悪くないか実際に確かめることが非常に重要です。 **「推測するな、計測せよ」** 的な発想です。DWHで実際何が起きているのかを理解することが最適化への第一歩となります。

:::message
この「コンパイル後のクエリと向き合う」という姿勢は incremental model 以外でも重要だと考えています。例えば dbt test は便利な機能ですが、実際に実行されているクエリがどのようなものか見たことあるでしょうか？設定によっては意味のないテストになっていたり、非効率なものになっていたりすることも見かけたことがあります。是非チェックしてみましょう。
:::

コンパイル後の実際に実行しているクエリと向き合うことが重要なのは分かったとして、どのようにコンパイル後のクエリを確認するのが良いでしょうか？いくつかパターンがあるかなと思うので紹介します。


### 実行履歴を確認する

一番シンプルなのはクエリの実行履歴を確認し、 run や build で実行されたクエリを DWH 側で直接確認する、という方法です。既に incremental model を運用している場合などではまずこれをやって見るのが良いでしょう。例えば Snowflake であれば Query History から Query Profile を確認し、非効率になっているクエリがないかを確認することが大事です。

また Snowflake であれば SELECT などのツールを利用すると dbt のモデルごとに Query History の情報をまとめて表示してくれるため非常に便利です。

![select-sample](/images/articles/dbt-high-performance-incremental-model/select-sample.png =500x)
*SELECTでdbtのintegrationの設定をした時の画面。詳細は https://select.dev/docs/dbt*

弊社の六車がSELECTについて紹介している記事もあるのでぜひご覧ください。
https://findy-tools.io/products/select/385/333


### 事前にクエリを生成する

実際に dbt run を実行するとなるとコストが高いといったことはよくあるため、事前にクエリを生成して、 CTAS/DELETE/INSERT など複数のステップの各クエリを確認できると便利なことは多いです。また事前にクエリを生成できれば explain で実行計画を確認することで、実際に実行せずとも partition pruning が効いているかどうか、確認することもできます。

事前にクエリを生成するとなるとまず先に浮かぶのは `dbt compile` でコンパイルしたクエリを確認するという方法でしょう。ただ、この方法は incremental model との相性が悪く、複数クエリからなる場合も最初に実行するクエリのみがコンパイルされそれしか確認できません。

そのため、複数ステップのクエリ全てを確認したい場合は `dbt run --empty` で empty flag を利用するのが良いと考えています。 empty flag は `limit 0` や `where false` などをよしなに差し込んで実行してくれるモードです。これによって既存のテーブルに影響を与えたり、重いクエリを実際に回したりせず、コンパイル後のクエリを確認することができます。

https://docs.getdbt.com/reference/commands/run#the---empty-flag


## incremental_predicates

`incremental_predicates` という config をご存知でしょうか？ config でこちらを設定しておくと、incremental model の各ステップのクエリで設定したsqlのステートメントが追加されるというものです。

https://docs.getdbt.com/docs/build/incremental-strategy#about-incremental_predicates

```sql
-- in models/my_incremental_model.sql

{{
  config(
    materialized = 'incremental',
    unique_key = 'id',
    cluster_by = ['session_start'],  
    incremental_strategy = 'merge',
    incremental_predicates = [
      "DBT_INTERNAL_DEST.session_start > dateadd(day, -7, current_date)"
    ]
  )
}}
...
```
と `incremental_predicates` を config で指定しておくと、以下のように明示的に絞り込むための条件が追加されるようになります。
```sql
merge into <existing_table> DBT_INTERNAL_DEST
    from <temp_table_with_new_records> DBT_INTERNAL_SOURCE
    on
        -- unique key
        DBT_INTERNAL_DEST.id = DBT_INTERNAL_SOURCE.id
        and
        -- custom predicate: limits data scan in the "old" data / existing table
        DBT_INTERNAL_DEST.session_start > dateadd(day, -7, current_date)
    when matched then update ...
    when not matched then insert ...
```

このように `incremental_predicates` を適切に使うと、各ステップでフィルタリングの条件を明示され partition pruning を確実に効かせる、といったことが可能になりパフォーマンス向上に利用できるでしょう。特に UUID のようなカーディなリティの高い列での突合が必要な場合に、追加で明示的な条件を追加することで高速化が見込めます。

今時のDWHは少ない where の条件からもよしなに最適化をしてくれることが多いですが、条件を明示することで確実に最適化を促すことが可能になるというイメージです。前節の `dbt run --empty` による事前のクエリ生成と組み合わせて、最適化が実際にできているかを確認しながらの利用がおすすめです。

:::message
現在の実装では `incremental_predicates` で直接 `ref` などを書き込むことはできないようです。そのため動的な条件にするためには以下のIssueにあるような工夫が必要となるのでこの点は注意が必要です。この辺りは今後使い勝手が良くなっていくことに期待しています。
https://github.com/dbt-labs/dbt-core/issues/6658
:::



## まとめ

dbt で大規模なデータを取り扱うためには必須の incremental model のプラクティスについて紹介してきました。

* partitioning を適切に行い効率的に差分を取り扱えるようにすることが大事
* コンパイル後の実際に実行されるクエリと向き合い、それを効率化するためにどのようにモデルやそのconfigを書き換えるべきかを考えることが大事
* `incremental_predicates` なども利用するとさらに最適化が可能かも

皆さんも incremental model に関する工夫があればぜひ教えてください！

