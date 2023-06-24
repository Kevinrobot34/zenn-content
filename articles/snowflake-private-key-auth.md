---
title: "Snowflakeで秘密鍵を環境変数に格納して認証する"
emoji: "❄️"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Snowflake", "認証", "dbt", "Terraform", "Security"]
published: true
---

Snowflake ではキーペア認証ができる。

https://docs.snowflake.com/ja/user-guide/key-pair-auth


dbt や Terraform などで Snowflake を組み合わせる際には当然認証情報を渡す必要がある。キーペア認証をする場合には基本的には

* private key そのもの
* private key のファイルのパス

のどちらかを渡すことになる。

例えば dbt コマンドを AWS ECS 上で実行するといったことを考えると、 private key そのものを Secrets として環境変数に設定しておき、それを利用してキーペア認証をするといったことがやりたくなる。しかし、 private key そのものを環境変数として格納して使う場合には、利用するライブラリによって渡し方を気をつけないといけないので、そのポイントを紹介する。


## 秘密鍵のフォーマット PEM について

PEM はいわゆる以下のような鍵などのファイルのエンコーディング形式のこと。

```key.pem
-----BEGIN PRIVATE KEY-----
MIIEvQIBADANBgkqhkiG9w0BAQEFAASCBKcwggSjAgEAAoIBAQCjffw/G1CpgJsP
ecZMT5b0KOo3dFLYn3jGgenIBjvJEEGGp6bQe4/AaZh5AmSxzi5MWYGiY7OEuPwz
...
ias/p13y8Pms8BI4c4fHqVM=
-----END PRIVATE KEY-----
```

詳細は [RFC 7468]( https://tex2e.github.io/rfc-translater/html/rfc7468.html ) で定義されている。

秘密鍵関連の細かい話は別の記事も書いているので参照してください。
https://zenn.dev/kevinrobot34/articles/private-key-summary


PEM形式の鍵を読み込むにはまずヘッダー `-----BEGIN{label}-----` とフッター `-----END{label}-----` を取り除き、64文字ごとに改行されている文字列を連結して、 Base64 でデコードする、という流れが必要になる。

なんらかのライブラリで秘密鍵をそのまま渡して Snowflake の認証をする際には、そのライブラリが上記の処理をやってくれるのかどうかで渡し方が変わってくる。つまり、

* PEM 形式の鍵をそのまま渡す
* ヘッダー・フッターを取り除き、改行も削除して `MIIE...` という一行の長い文字列にして（ Base64 でデコードする直前まで整形して）渡す

のどちらかに基本的にはなる。


## dbt-snowflake での秘密鍵の取り扱い

dbt-snowflake の `private_key` の取り扱いを見てみるとこんな感じ。

https://github.com/dbt-labs/dbt-snowflake/blob/e1cde45c8b8ee7b4862e1a3c6417ad4f4e6df4b2/dbt/adapters/snowflake/connections.py#L245-L250

`private_key` を直接 base64 でデコードしている。
よって、「ヘッダー・フッターを取り除き、改行も削除して `MIIE...` という一行の長い文字列」を `private_key` として渡す必要がある。


## terraform-provider-snowflake での秘密鍵の取り扱い

terraform-provider-snowflake の `private_key` の取り扱いを見てみるとこんな感じ。

https://github.com/Snowflake-Labs/terraform-provider-snowflake/blob/98bbf2c83e804b69ad2b62ea4862e6328a8494e6/pkg/provider/provider.go#L513-L517

`private_key` を標準ライブラリの `encoding/pem` の Decode 関数を使って読み込んでいる。
https://pkg.go.dev/encoding/pem#example-Decode

よって、 PEM 形式の鍵をそのまま `private_key` として渡せば良い。


## snowflake-connector-python での秘密鍵の取り扱い

snowflake-connector-python でも `private_key` は利用できる。公式ドキュメントには秘密鍵のファイルパスを渡して読み込む例が記載されている。
https://docs.snowflake.com/ja/developer-guide/python-connector/python-connector-example#using-key-pair-authentication-key-pair-rotation


実際に認証する部分を確認するとこんな感じ。

https://github.com/snowflakedb/snowflake-connector-python/blob/48d8d3fe9c69f7941797654480ac174e578bcb03/src/snowflake/connector/auth/keypair.py#L106-L110

`private_key` は bytes で渡さないといけないことがわかるので、自分で base64 でデコードした上で渡す必要がある。

例えば `export SNOWFLAKE_PRIVATE_KEY='MIIE....'` と秘密鍵を一行にして環境変数に設定しておき、以下のようなコードを書けば認証が通る。

```python
import base64
import os

import snowflake.connector

SNOWFLAKE_ACCOUNT = os.environ['SNOWFLAKE_ACCOUNT']
SNOWFLAKE_PRIVATE_KEY = os.environ['SNOWFLAKE_PRIVATE_KEY']
SNOWFLAKE_USER = os.environ['SNOWFLAKE_USER']

ctx = snowflake.connector.connect(
    account=SNOWFLAKE_ACCOUNT,
    user=SNOWFLAKE_USER,
    private_key=base64.b64decode(SNOWFLAKE_PRIVATE_KEY),
)

with ctx.cursor() as cs:
    cs.execute("SELECT current_version()")
    one_row = cs.fetchone()
    print(one_row[0])
    # 7.21.1
```


## まとめ

キーペア認証で秘密鍵を直接渡したいときには、各ライブラリの実装によって渡すべきものが変わるので注意しましょう。