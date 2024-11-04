---
title: "Snowflakeのパスワードの有効期限切れの罠"
emoji: "👻"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Snowflake", "Authentication", "Security", "DataEngineering"]
published: false
publication_name: dataheroes
---

## はじめに

こんにちは！ナウキャストのデータエンジニアのけびんです。

Snowflake では認証方法として

* Password
* Key Pair
* SAML を利用した SSO
* OAuth

が用意されています。

あるユーザーでの認証方法としてこのうちの複数を利用しているケースはよくあるのではないでしょうか？

特にパスワード認証と他の認証方法を組み合わせた際に、パスワードの有効期限が切れてしまうとパスワードの変更するまで**任意の認証方法で認証ができなくなる**という仕様があります。


## パスワードとキーペア認証

パスワードとキーペア認証の両方を利用している場合を考えましょう。
Snowsight にはパスワード認証を利用し、開発時にはキーペア認証を利用するというような使い分けをしているようなイメージです。

この時、パスワードにはパスワードポリシーによって有効期限が設定されているとします。
https://docs.snowflake.com/ja/user-guide/admin-user-management#label-using-password-policies


この時、パスワードの有効期限が切れてしまうと、**キーペアで認証を行おうとしても以下のようにパスワードの変更を求めるエラーが出てしまいます。**
```
250001 (08001): Failed to connect to DB: abc123xy9-sample_account..snowflakecomputing.com:443. Specified password has expired. Password must be changed using the Snowflake web console.
```


## パスワードとSSO

パスワード認証とSSOを組み合わせている場合でも同様です。
一度パスワードが期限切れとなると、SSOでログインしようとした場合もパスワードの変更を求めるエラーが出てしまいます。

https://community.snowflake.com/s/article/Snowflake-Password-has-Expired-using-Single-Sign-On-SSO-authentication


## 対応策

このような状況に陥ってしまった場合には、対策は２つです。


### パスワードを変更する

パスワード認証の必要がある場合にはパスワードを変更しましょう

* Snowsight からパスワードを変更する
* SQL で設定する
    ```sql
    ALTER USER user1 SET PASSWORD='H8MZRqa8gEe/kvHzvJ+Giq94DuCYoQXmfbb$Xnt' MUST_CHANGE_PASSWORD = TRUE;
    ```

ユーザー自身ではなく管理者が上記の対応をしないといけない場合には `MUST_CHANGE_PASSWORD` を True にしておくと良いでしょう。


### パスワードの利用を停止する

そもそもパスワード認証を実はもう使っていないというパターンもあると思います。
その場合にはパスワードをNullに設定し、そもそも他の認証方法だけを利用するようにしましょう。

権限があるロールで以下を実行すればOKです
```sql
ALTER USER user1 SET PASSWORD=NULL;
```

システムから利用するユーザーの場合、キーペア認証だけあれば良いということも多いと思いますので、適宜パスワードはNullにしておきましょう。

Authentication Policy も適宜組み合わせることでより安全になるはずです。
https://docs.snowflake.com/ja/user-guide/authentication-policies


## まとめ


* パスワード認証と他の認証方法を組み合わせた場合、任意の認証方法利用時にパスワードの有効期限切れのエラーは出てしまう
* パスワードを更新するか、パスワードをNullにすれば対応可能
* そもそもパスワードが必要なければ Authentication Policy なども利用すると安全性が上がる