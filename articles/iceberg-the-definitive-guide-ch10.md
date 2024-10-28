---
title: "Apache Iceberg: The Definitive Guid 10章 Apache Iceberg in Production"
emoji: "🧊"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Iceberg", "DataEngienering", "OTF", "Snowflake"]
published: true
publication_name: dataheroes
---

## はじめに

こんにちは！ナウキャストでデータエンジニアをしているけびんです。

SnowVillage で行っている [Apache Iceberg: The Definitive Guid]( https://www.oreilly.com/library/view/apache-iceberg-the/9781098148614/ ) 輪読会の10章後半の資料です。10章は "Apache Iceberg in Production" というタイトルで Iceberg を本番で運用する際に大事なポイントなどが書かれていますが、その内容をまとめたり、調べてみつけた関連した内容も適宜記載したりしています。

:::message
この形式が私の感想・コメントです。
:::


## 10章の introduction

データエンジニアは効率的で信頼性が高くそして安全な方法でデータを収集・保管・処理することに責任を持ちます。データを本番で取り扱う際には様々なプラクティスに従い、データが正確で不整合なく、そしていつでもアクセス可能な状態を保つことが必要です。

この章では Iceberg table を本番運用する際に監視したりメンテナンスするのに役立つツールを見ていきます。具体的には以下の４つが紹介されています。

* Metadata Tables
* Branching and Tagging
* Multi-table Transactions
* Rollback


これらの各種の方法は reactive （事後的）にも proactive （予防的）にも利用することができる。
* **reactive** approach
  * 問題が起き次第、事後的に対処するアプローチ
  * Iceberg の場合の具体例
    * 大きくなりすぎたパーティションを書き換える
    * 悪いデータが取り込まれてしまったテーブルを roll back する
* **proactive** approach
  * 事前に対処し、そもそも問題が起きないように予防するアプローチ
  * Iceberg の場合の具体例
    * metadata table を利用し、パフォーマンスに影響が出る前からパーティションのサイズを監視しておく
    * branch を利用し、悪いデータが直接本番のテーブルに取り込まれないような運用をする

なるべく proactive に動いて様々な問題を予防することも大事ですが、reacitve な方法も適切に知っておき何か問題が起きた時に迅速に対応できるようになっておくことももちろん大事です。

早速それぞれの機能について具体的に見ていきましょう。


## Metadata Tables

### Metadata Tables の仕組み

Iceberg の最も強力な特徴の一つは様々なメタデータを適切に保持する Table Format であることです。
Data files と Metadata files を分離し、適切な階層構造で管理することで様々なメリットを生み出している、ということをたくさんこの本でも見てきました。

階層構造で管理されている Metadata files はその名の通り様々なメタデータを保持しており、これらの情報は Iceberg Table を運用する際に適宜参照すると非常に有用です。

Iceberg ではこれらのメタデータを**標準的な SQL で**参照できるように Metadata Tables が定義されています。

![iceberg-metadata-tables](/images/articles/iceberg-the-definitive-guide-ch10/iceberg-metadata-tables.png)
*https://www.apachecon.com/acna2022/slides/02_Ho_Icebergs_Best_Secret.pdf より*


上図のように、 Metadata Table ごとに Metadata Files の階層のうちどこに含まれる情報なのかが定義されていて、よしなに必要な情報を抽出してくれるようになっています。



### 各種 metadata table の解説

Iceberg で定義されている metadata tables は以下に列挙されています。
https://iceberg.apache.org/docs/latest/spark-queries/#inspecting-tables

各テーブルの中身については 10章前半として発表していただいた Doi さんの資料の中で解説されているので、こちらをご参照ください。

**後で資料のリンクを貼る**



## Branching and Tagging

ソフトウェア開発においてコードを Git でバージョン管理し、 branch や tag を利用した運用を行うというのは現在当たり前になっています。
同様の方法でデータを管理することも有用だと考えられています。Iceberg ではテーブルの Snapshot が Git でいう commit のような役割を果たしており、テーブルに対する branch や tag の作成がサポートされています。

branch を作ってテーブルに対する複数の作業を並行して行ったり、あるブランチでの開発が完了し各種データの検証などもパスしてから本番環境のブランチにマージする、といったことが可能になります。また tag として snapshot に名前をつけ、その snapshot にアクセスしやすくしたり保護したりすることもできます。



### Table-level tagging

https://iceberg.apache.org/docs/latest/branching/

上記のドキュメントでは Branching や tagging のユースケース、使い方が紹介されています。

一つ目のユースケースとして監査のために Historical tags を作成し、保持期間などを適切に設定する事例が紹介されています。

```sql
-- Create a tag for the first end of week snapshot. Retain the snapshot for a week
ALTER TABLE prod.db.table CREATE TAG `EOW-01` AS OF VERSION 7 RETAIN 7 DAYS;
-- Create a tag for the first end of month snapshot. Retain the snapshot for 6 months
ALTER TABLE prod.db.table CREATE TAG `EOM-01` AS OF VERSION 30 RETAIN 180 DAYS;
-- Create a tag for the end of the year and retain it forever.
ALTER TABLE prod.db.table CREATE TAG `EOY-2023` AS OF VERSION 365;
-- Create a branch "test-branch" which will be retained for 7 days along with the  latest 2 snapshots
ALTER TABLE prod.db.table CREATE BRANCH `test-branch` RETAIN 7 DAYS WITH SNAPSHOT RETENTION 2 SNAPSHOTS;
```
![table-level-tagging-historical-snapshot-tag](/images/articles/iceberg-the-definitive-guide-ch10/table-level-tagging-historical-snapshot-tag.png)
*https://iceberg.apache.org/docs/latest/branching/#historical-tags より*



tag を使うことで特定のスナップショットに簡単にアクセスできるようになるだけでなく、
保持期間などを明示的に設定することができスナップショットのライフサイクル管理にもなります。



### Table-level branching

同じドキュメントの中で branching のユースケースも紹介されています。

```sql
-- 1. First ensure write.wap.enabled is set.
ALTER TABLE db.table SET TBLPROPERTIES ('write.wap.enabled'='true');
-- 2. Create audit-branch starting from snapshot 3, which will be written to and retained for 1 week.
ALTER TABLE db.table CREATE BRANCH `audit-branch` AS OF VERSION 3 RETAIN 7 DAYS;
-- 3. Writes are performed on a separate audit-branch independent from the main table history.
SET spark.wap.branch = audit-branch
INSERT INTO prod.db.table VALUES (3, 'c');

-- 4. A validation workflow can validate (e.g. data quality) the state of audit-branch.
-- execute some validation query

-- 5. After validation, the main branch can be fastForward to the head of audit-branch to update the main table state.
CALL catalog_name.system.fast_forward('prod.db.table', 'main', 'audit-branch');
-- 6. The branch reference will be removed when expireSnapshots is run 1 week later.
```
![table-level-branching-audit-branch](/images/articles/iceberg-the-definitive-guide-ch10/table-level-branching-audit-branch.png)
*https://iceberg.apache.org/docs/latest/branching/#audit-branch より*


いきなり本番環境にデータの追加を行うのではなく、 audit-branch に一旦取り込み、そこでデータの検証などが完了してから fast-forward マージする形で本番環境にデータを取り込む、というイメージです。


:::message
本番テーブルのデータ品質を保つためには非常に便利な機能だなと思う一方で、この audit-branch のシンプルなブランチ運用くらいしか回らないのではないか、という気持ちにはなりました。
乱用すると branch が管理しきれなくなって大変なことになりそうなので運用ルール決めが大事なのかもしれません。
:::


### Catalog-level branching and tagging

Table-level での branch や tag の作成は任意のカタログでサポートされていますが、
Nessie カタログを利用すると Catalog レベルでの branch や tag の作成が可能になります。

Nessie をカタログでは、データレイク全体を一つのエンティティとして取り扱い、複数テーブルに跨った変更を追跡することが可能です。

基本的なメリットは table-level と変わりませんが、大規模なカタログの場合、個々のテーブルでバージョン管理するよりもカタログレベルで管理した方が多数のテーブルを同時に処理できるためデータ管理プロセスが大幅に簡素化されるでしょう。



## Multi-table Transactions

Multi-table Transaction は複数テーブルにまたがる操作の "consistency" と "isolatioon" を実現するために重要です。

Multi-table transaction では、複数の操作（これは複数のテーブルにまたがる場合もある）が一つのアトミックな作業の単位として扱われます。これはつまりこれらの複数の操作が成功するか、もしくはいずれかが失敗した場合には全ての変更がロールバックされ元に戻るということを意味します。


### Consistency

Multi-table transaction はデータの整合性を維持するために非常に重要で、DBMSの重要な側面の一つです。

具体例を考えましょう。
例えばサプライチェーン管理システムがあり注文を管理する `Order` テーブルと、在庫を管理する `Inventory` テーブルがあるとする。注文があると `Order` が更新されると共に、 `Inventory` の該当する在庫のレコードは減らされます。
もしこの２つの操作が一つのトランザクションとして取り扱われなかった場合、 `Order` は更新されるが、 `Inventory` だけロールバックされ在庫の減少がデータベースに反映されない、みたいなことが起こり得ます。
逆に、２つの操作が一つのトランザクションとして取り扱われると、 `Order` の更新は成功して `Inventory` の更新は失敗した場合、`Inventroy`も`Order`もどちらもロールバックされて元の状態に戻る。こうしてデータの整合性は保たれるわけです。

このように複数テーブルにまたがった consistency を維持するためには Multi-table transaction は重要な役割を果たします。


### Isolation

同時に複数のトランザクションが実行された際にお互いに干渉しない、という性質。
それぞれのトランザクションはお互いのことを認識せず、 dirty read や data corruption といったことは起きないことが保証されます。

この辺りは基本的に Single-table Transaction でも話は一緒なので、そちらの事例なども参考になるかもしれません。
https://medium.com/@tglawless/apache-iceberg-acid-transactions-ec9d7b7afff5


### 現状

10章で触れられているものの、2024年10月末現在、 Apache Icebrg としてはまだ Multi-table Transaction は開発中です。関連資料は以下。

* [apache/iceberg: Add Multi-Table Transaction API #10617]( https://github.com/apache/iceberg/issues/10617 )
* [DesignDocs - Multi-Table Transactions in Iceberg]( https://docs.google.com/document/d/1UxXifU8iqP_byaW4E2RuKZx1nobxmAvc5urVcWas1B8/edit?tab=t.0 )


カタログが Multi-table transaction を制御するような構成が提案されているようです。

:::message
今までの他の章・節でも見てきたように、 Iceberg を運用する際にはカタログがいろんな面で重要な役割を果たすことが分かります。
:::



## Rollback


Rollback は undo ボタンのようなもので、テーブルが望ましく無い状態になってしまった時にテーブルを望ましい元の状態に戻すための機能です。Reactive に対応するために重要になります。

### Rolling Back at the Table Level

Apache Iceberg ではあるテーブルの状態を切り戻すために以下の４つの Spark Procedure を用意されています。
ロールバックするスナップショットの指定の仕方がいろいろあるという感じです。最初に紹介した metadata tables は、どのスナップショットにロールバックすべきかを調査するために非常な有用なツールとなるので、適宜組み合わせながら使いましょう。

* [`rollback_to_snapshot`]( https://iceberg.apache.org/docs/1.5.1/spark-procedures/#rollback_to_snapshot )
    * snapshot ID を指定してロールバックするプロシージャー
    * 例： `db.sample` テーブルの snapshot ID `1` にロールバックする場合
        ```sql
        CALL catalog_name.system.rollback_to_snapshot('db.sample', 1);
        ```
    * このプロシージャが実行されると、対象のテーブルが指定された snapshot id を参照する状態になる
    * この際に metadata やデータファイルそしてスナップショットは変わらない
    * 他の操作と同時に実行されても安全なように設計されている
    * 疑問：では結局何が変わっているの？
        * 実装は [`RollbackToSnapshotProcedure`]( https://github.com/apache/iceberg/blob/2b55fef7cc2a249d864ac26d85a4923313d96a59/spark/v3.5/spark/src/main/java/org/apache/iceberg/spark/procedures/RollbackToSnapshotProcedure.java#L42 ) らしいけど分からん
        * 予想としては metadata file の `current-snapshot-id` が更新されて、  `snapshot-log` も適宜更新される、というもの
* [`rollback_to_timestamp`]( https://iceberg.apache.org/docs/1.5.1/spark-procedures/#rollback_to_timestamp )
    * `rollback_to_snapshot` と基本的に同じだが、 snapshot ID ではなくタイムスタンプでロールバック先を指定する
* [`set_current_snapshot`]( https://iceberg.apache.org/docs/1.5.1/spark-procedures/#set_current_snapshot )
    * `rollback_to_snapshot` に近いが、このプロシージャの場合には、現在のテーブルの状態の先祖の snapshot である必要はなく、別のブランチやタグにある利用可能な任意の snapshot を設定することが可能
* [`cherrypick_snapshot`]( https://iceberg.apache.org/docs/1.5.1/spark-procedures/#cherrypick_snapshot )
    * まさに Git の cherry-pick のような処理を行うためのもの
    * メタデータのみの操作であり、別のスナップショットからの変更を組み込んだ新しいスナップショットを生成する
        * データファイルは作成されない


これらのプロシージャを適切に利用することで、Icebergで管理しているデータを Git Like に管理することが可能になります。

:::message alert
進んだ注意
* これらの rollback 処理は、キャッシュされた全ての Spark プランを無効にし、後続の操作がテーブルの更新された状態を利用するようにする
* これらのプロシージャを利用するためには、テーブルに対するこれらの操作を実行する権限を持っている必要がある
:::


### Rolling Back at the Catalog Level

Nessie を Apache Iceberg のカタログとして利用することの利点の一つは、カタログレベルでのロールバックが可能になります。

GitHub のようなバージョンコントロールシステムを利用することでソフトウェアエンジニアがコードベース全体を以前のバージョンに戻せるように、Nessie はデータエンジニアらがデータベース全体を以前の環境に戻すことを可能にします。。

具体例として、あるバッチ処理を実行し様々なテーブルに変更を加えた後、その処理に誤りがあったことに気付いた場合を考えましょう。
Iceberg はテーブルレベルのロールバックをサポートしているので、テーブルを一つずつロールバックすることで元の状態を復元することができます。しかし Nessie を利用している場合にはカタログレベルでロールバックすれば全てのテーブルを瞬時に以前の状態に戻すことが可能です。

このようにワークロードが複雑で複数のテーブルを取り扱っているような場合にはカタログレベルでのロールバックが有効な Nessie は便利でしょう。

:::message
Nessie は Catalog-level の branching / tagging も Rollback も対応しており、規模が大きいユースケースにマッチすることが多そうな予感です。
:::


## まとめ


* Apache Iceberg の "Metadata Tables " / "Branching and Tagging" / "Multi-table Transaction" / "Rollback" の４つの機能はどれも Iceberg を本番運用する上で重要な役割を果たすことをみてきた
    * Metadata Table は Iceberg で管理されたテーブルに関する様々な情報を提供し、 reactive / proactive なアプローチ両方において非常に重要となる
    * Branching and Tagging を適切に利用することで本番環境を破壊することなく変更を検証する環境を提供するという proactive な管理が強化される
    * Multi-table Transaction を利用することで複数テーブルにまたがる処理の consistency / isolation が実現可能になる予定
    * Rollback の機能は障害発生時に簡単に回復する手段を提供しており、 reactive な対応が強化される
* 次の章では Streaming data の取り扱いについて見ていく
