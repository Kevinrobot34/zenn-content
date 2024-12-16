---
title: "SnowflakeのJoinを理解する"
emoji: "😺"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Snowflake", "SQL", "DataEngineering"]
published: false
publication_name: finatext
---

この記事は、[Snwoflake Advent Calendar 2024]( https://qiita.com/advent-calendar/2024/snowflake )の16日目の記事です。

## はじめに

こんにちは！ナウキャストのデータエンジニアのけびんです。

Snowflake が Join の仕方を工夫することでパフォーマンスを改善したという以下の記事を先日読み、面白かったので、本記事ではこれについて解説しようと思います。
https://www.snowflake.com/engineering-blog/query-acceleration-smarter-join-decisions/

Join の仕方について解像度高く理解しておくと、クエリの最適化にも役立つはずです！

:::message
自分もこの領域の専門家というわけではないので、もし間違いがあれば指摘していただけると幸いです。
:::



## RDB における Join の方法

いきなり Snowflake の Join にいく前にまずはリレーショナルデータベースにおける Join のアルゴリズムについて確認しておきましょう。検索するとさまざまな資料があるかと思いますが、以下の記事が分かりやすいです。

https://zenn.dev/loglass/articles/84f15be9a4d2c9


### Nested Loop

単純に二重ループを回して結合する方法です。テーブルAの各レコードをテーブルBの全レコードと比較して条件に一致するものを探すというようなナイーブな方法で、コストは二つのテーブルのレコード数の**積**に比例します。
検索するテーブルBにindexがあると速くなったりします。

https://en.wikipedia.org/wiki/Nested_loop_join


### Sort Merge

２つのテーブルを事前にソートしておくことで、Nested Loop よりも計算量を削減をしているのが Sort Merge です。事前にソートしておくことで、２つのテーブルのレコードを上から順番に捜査し、 Join を行います。事前にソート済みであれば、このコストは二つのテーブルのレコード数の和に比例します。

https://en.wikipedia.org/wiki/Sort-merge_join

### Hash

Hash Join は前の２つとは少し異なるアプローチで結合を行います。具体的には以下の２つのStepで実行されます。

* **Build フェーズ**：ハッシュテーブルの準備
  * 片方のテーブルを利用してハッシュテーブルを作る
  * 基本的には小さい方のテーブルを利用する
    * ハッシュテーブルを構築するのに使うテーブルのことをビルド側と言ったりする
  * ハッシュテーブルは join に利用するカラムを他の残りのカラムにマッピングするようなイメージ
* **Probe フェーズ**：レコードの突合
  * ハッシュテーブル構築後、もう一方のテーブルをスキャンし、ハッシュテーブルからビルド側の対応する行を探す
    * こちらのテーブルはプローブ側と言ったりする

この方法はテーブルがソート済みかに関係なく、コストは２つのテーブルのレコード数の和に比例します。

https://en.wikipedia.org/wiki/Hash_join


データベースの種類によって対応状況が異なっていたり、indexや他の方法と組み合わされていたりといった違いがあったりするので詳細は各DBの join アルゴリズムに関するドキュメントなどを参照するのが良いでしょう。


## Snowflake における Join

一般的なOLTPと違い、SnowflakeなどのOLAPでは

* 対象とするデータサイズがかなり大きい
* 複数Nodeで並列分散処理を行う

といったポイントがあるでしょう。

Snowflake では大規模なデータに対するクエリが多いため、 Hash Join が利用されることが多いはずですが、その Hash Join にも Broadcast Join と Hash-Hash Join の２つの方法が用意されています。


### Broadcast Join


![snowflake-join-broadcast](/images/articles/snowflake-join-strategy/snowflake-join-broadcast.png =500x)
*https://www.snowflake.com/engineering-blog/query-acceleration-smarter-join-decisions/ より*


### Hash-Hash Join

![snowflake-join-hashhash](/images/articles/snowflake-join-strategy/snowflake-join-hashhash.png =500x)
*https://www.snowflake.com/engineering-blog/query-acceleration-smarter-join-decisions/ より*



## Join の最適化のポイント

* Join Order
* Join Method

https://en.wikipedia.org/wiki/Join_(SQL)#Implementation



## Snowflake の Adaptive Join Decisions



## まとめ

