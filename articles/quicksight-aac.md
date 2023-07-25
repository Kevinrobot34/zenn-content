---
title: "Quicksight の Asset As Code"
emoji: "👏"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: []
published: false
---

## Asset As Code とは


## AaC の対象


## AaC の流れ


## ユースケース

* Analysi や Dashboard をブラウザから作成し、その定義を JSON/YAML として APIで 取得することでそのスナップショットを取っておく
  * また定義は JSON/YAML なのでこれを Github などでバージョン管理する
  * バージョン管理すると Analysis や Dashboard のロールバックもしやすくなる
* AWSアカウントを跨いで Analysis や Dashboard を共有する
  * Analysis や Dashboard の定義を適宜少し変更すれば開発環境で作った Asset を本番環境にデプロイする、といったことが容易にできるようになる


## AaC VS. IaC


## References