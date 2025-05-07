---
title: "Snowflake Terraform Provider のバージョンアップ Tips"
emoji: "💨"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Snowflake", "Terraform", "DataEngineering"]
published: false
publication_name: finatext
---

## はじめに

こんにちは！ナウキャストのデータエンジニアのけびんです。

2025年4月23日に terraform-provider-snowflake の 2.0.0 がリリースされ、晴れて GA となりましたね！これによりサポートも対応してくれるようになるということで、今後 Terraform を利用した Snowflake のインフラ管理がよりやりやすくなっていきそうだなと感じています。

しかしそもそも 2.0.0 にまでバージョンを上げることがなかなか大変です。自分もかなり苦労しつつ、なんとか最新バージョンを利用できる状態まで辿り着きました。

本ブログでは、 provider version up の際に知っておくと便利なことを紹介していこうと思います。



## 基本的な進め方

基本的な進め方は以下の通りになるかなと思います。


* version のリリースノートと migration guide を読む
  * https://github.com/snowflakedb/terraform-provider-snowflake/releases
  * https://github.com/snowflakedb/terraform-provider-snowflake/blob/main/MIGRATION_GUIDE.md
* とりあえずバージョンを変更して plan してみる
  * plan するだけであれば影響はないのでとりあえずやってみるのはおすすめ
  * 出てきた差分について、改めて上記の資料をよく読む
* TFの変更をする
  * Plan時の差分がなくなるまで適宜変更する
  * しかし実際には完全に差分がなくすことは難しかったりする
  * Destroy は最低限出ていないようにする



## import / removed block の活用

* コマンドで import や state rm をやってもいいし場合によってはそうするしかない
* が、しかしリソースが多いとコマンドで一つ一つやるのはめっちゃ時間がかかる
* また、 state を操作する系のコマンドは複数人開発の時に事故につながりやすい
* block で記載しまとめて対応できるようにするとハッピー
* import block は for_each が使えるが、 removed は for_each が使えないのでその点は注意
* removed で for_each は使えないが、 module などであればまとめて remove することは一応できる
* removed は module の中に記載できるが、 import はできない

### 書き方の具体例



## LLM の活用

* ざっくりLLMにバージョンアップのための変更やって、とか依頼すると失敗する
* 直接 LLM に TF ファイルを変更したり、 import / removed ブロックを全て書いてもらうとかは意外と失敗する
  * Provider のバージョン変更による破壊的変更が多すぎて直接うまくやるのは難しい感じ
* なので、 provider version update するために必要な作業をやるスクリプトを書いてもらう、みたいなやり方をすると良い
* もしくは具体的に import block をこうやって書いて、みたいなやり方はうまくいきやすい
  * https://zenn.dev/fap/articles/5272be5b55dc11