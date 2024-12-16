---
title: "Snowflake の Adaptive Join Decisions"
emoji: "🌳"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Snowflake", "SQL", "DataEngineering"]
published: true
publication_name: finatext
---

この記事は、[Snwoflake Advent Calendar 2024]( https://qiita.com/advent-calendar/2024/snowflake )の16日目の記事です。

## はじめに

こんにちは！ナウキャストのデータエンジニアのけびんです。

Snowflake は日々クエリ実行が早くなるようにさまざまな開発をしてくれています。最近新しいパフォーマンス機能である **"Adaptive Join Decisions"** がリリースされ、 **9.5%** もクエリパフォーマンスが改善されたそうです。ユーザーは自動的にこの恩恵を受けられます、嬉しいですね。
https://www.snowflake.com/engineering-blog/query-acceleration-smarter-join-decisions/

この詳細が面白かったので、本記事ではこれについて解説しようと思います。 Snowflake が裏側でどのように Join を取り扱っているのかを理解しておくと、データエンジニアらがクエリの最適化する際にも役立つのではないかと思うので、ぜひご覧ください！



## RDB における Join の方法

いきなり Snowflake の Join にいく前にまずはリレーショナルデータベースにおける Join のアルゴリズムについて確認しておきましょう。検索するとさまざまな資料があるかと思いますが、以下の記事が分かりやすいです。

https://zenn.dev/loglass/articles/84f15be9a4d2c9


### Nested Loop

単純に二重ループを回して結合する方法です。テーブルAの各レコードをテーブルBの全レコードと比較して条件に一致するものを探すというようなナイーブな方法で、コストは二つのテーブルのレコード数の**積**に比例します。検索するテーブルBにindexがあると速くなったりします。

https://en.wikipedia.org/wiki/Nested_loop_join


### Sort Merge

２つのテーブルを事前にソートしておくことで、Nested Loop よりも計算量を削減をしているのが Sort Merge です。事前にソートしておくことで、２つのテーブルのレコードを上から順番に走査し、 Join を行います。事前にソート済みであれば、このコストは二つのテーブルのレコード数の**和**に比例します。

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

この方法はテーブルがソート済みかに関係なく、コストは２つのテーブルのレコード数の**和**に比例します。

https://en.wikipedia.org/wiki/Hash_join


データベースの種類によって対応状況が異なっていたり、indexや他の方法と組み合わされていたりといった違いがあったりするので詳細は各DBの join アルゴリズムに関するドキュメントなどを参照するのが良いでしょう。


## Snowflake における Join

一般的なOLTPと違い、SnowflakeなどのOLAPでは

* 対象とするデータサイズがかなり大きい
* 複数Nodeで並列分散処理を行う

といったポイントがあるでしょう。Snowflake の Hash Join では Broadcast Join と Hash-Hash Join の２つの方法が用意されています。


### Broadcast Join

Broadcast Join は Build フェーズで構築したハッシュテーブルを全てのワーカーノードに配っておき、各ノードで Probe フェーズの突合を行なっていく、という方法です。

ビルド側のテーブルが大きい場合には各ノードにハッシュテーブルを配る部分がオーバーヘッドになったりメモリに乗り切らずにパフォーマンスが出ないということがありえますが、ビルド側のテーブルが小さい場合には並列分散処理の恩恵を受けやすいです。

![snowflake-join-broadcast](/images/articles/snowflake-join-strategy/snowflake-join-broadcast.png)
*https://www.snowflake.com/engineering-blog/query-acceleration-smarter-join-decisions/ より*


### Hash-Hash Join

Hash-Hash Join (もしくは Hash-Partitioning Hash Join) は大規模なデータに適した方法です。まず最初に Build 側も Probe 側もハッシュ関数を利用してパーティショニングしておきます。それぞれ対応する一部のパーティションをワーカーノードに配置し、その中で Hash Join を行う、というようなイメージです。

このようにハッシュ関数によってうまくデータを細かく分割しそれを上手く配布することで並列分散処理の恩恵を大規模データに対する Join でも得られるようにしています。

![snowflake-join-hashhash](/images/articles/snowflake-join-strategy/snowflake-join-hashhash.png)
*https://www.snowflake.com/engineering-blog/query-acceleration-smarter-join-decisions/ より*



### Join の最適化のポイント

今までの話を踏まえると分かりやすいかと思いますが、 Join を最適化するためには以下の２つのポイントについて適切に設定することが重要になります。

* Join Order
  * 関係代数としての結合の演算は順序を入れ替えても最終結果は変わらないです
  * しかし例えば Hash Join の結合の処理は Build 側と Probe 側とで非対称なため、順序を変えることでパフォーマンスに大きな影響があり得ます
* Join Method
  * Join のアルゴリズムはどれを使っても最終結果は変わらないです
  * しかしどの方法が最適かは対象のテーブルのサイズやフィルター条件など、さまざまな要因によって変わり得ます
  * 例えば以下のように Build 側のテーブルのサイズによって Broadcast Join が良いか Hash-Hash Join が良いかは変わります
    ![snowflake-join-comparison](/images/articles/snowflake-join-strategy/snowflake-join-comparison.png)
    *https://www.snowflake.com/engineering-blog/query-acceleration-smarter-join-decisions/ より*


https://en.wikipedia.org/wiki/Join_(SQL)#Implementation



## Snowflake の Adaptive Join Decisions

多くのシステムではコンパイル時に Join の順序や方法を決定するのに対し、 Snowflake はクエリ実行時に適当的にこれを決定するアプローチが取られており、　"Adaptive Join Decisions" として冒頭の記事で紹介されています。要はコンパイル時には得られない情報を適切に利用することで、 Join に関する決定をより最適なものにしようというイメージです。

おそらく "Adaptive Join Decisions" の中でもさまざまな工夫があるはずですが、この記事では "probe-side annotated join decision-making" という改善について触れられています。具体的には、 runtime 実行時のビルド側テーブルのサイズ推定と、プローブ側のコンパイル時のアノテーション情報を組み合わせるというものです。これにより高コストな Broadcast Join を避けられるようになった、という以下の事例が紹介されています。

* Right-deep join trees / Coordinated join decisions
  * OLAP では単一の大きい fact テーブルに対し、複数の dim テーブルを結合していくというユースケースがよくあり、このような場合には右に深くなるような Join Tree となるように結合処理を行っていくのが良いとされています
  * これにより中間テーブルを保存することを避け、前の結合処理の結果をそのまま次でも利用することができ、高速になります
  * またこの複数の join の中でどれで Broadcast Join を利用しどれで Hash-Hash Join を利用すべきかも動的に判断し決定するようになっています
    ![sample-join-tree](/images/articles/snowflake-join-strategy/sample-join-tree.png =500x)
    *https://www.snowflake.com/engineering-blog/query-acceleration-smarter-join-decisions/ より*

このようにコンパイル時ではなく、クエリ実行時の情報を踏まえた adaptive なアプローチにより、Snowflakeはデータサイズや特性に動的に適応する効率的なクエリ処理を実現しているようです。


## まとめ

Snowflake の 9.5% ものパフォーマンスの向上の立役者である "Adaptive Join Decisions" について見てきました。これ自体は勝手に適用されて我々 Snowflake ユーザーはその恩恵にあずかれるので非常にありがたいですね。またこの内容を理解しておくことで、自分のクエリを直接改善するためのヒントも得られる部分はあるのではないかと思っています。何らかの形で最適化のヒントにつながらないか、ぜひ考えてみてください！