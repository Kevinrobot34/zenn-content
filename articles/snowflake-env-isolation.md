---
title: "Snowflakeでの環境分離について考える"
emoji: "❄️"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Snowflake"]
published: false
---


最近 Snowflake 関係で登壇したり、イベントでいろんな方と話す中で、「Snowflake で環境分離をどうやるか？」というトピックが出てきたので、自分の考えを整理してみる。

## 論点



## 分離方法

### Account で分離する

* Pros
  * 確実に環境を切り分けることができる
* Cons
  * 同一アカウントであれば使える機能が使えない
  * Organization の設定等をしていないと請求や諸々の管理が面倒かも(?)

### Database で分離する

* Pros
  * 環境間のデータのやり取りが容易
    * 本番環境のデータを開発環境にクローンしてから新規モデルの検証をするといったことがやりたい場合はあると思う
* Cons
  * 権限管理をしっかりやらないと、意図せず本番環境に何かやってしまうということが起こり得る


## References

* [How to set up development and production environments in Snowflake]( https://www.propeldata.com/blog/how-to-set-up-development-and-production-environments-in-snowflake )