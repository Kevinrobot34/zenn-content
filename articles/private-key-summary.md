---
title: "秘密鍵のファイル周りの話"
emoji: "🔑"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Security", "公開鍵暗号", "openssl", "rsa", "pem"]
published: true
---


開発していると、公開鍵暗号の技術を利用する場面は多々ある。

* EC2インスタンスへのSSHする時や、Snowflakeの認証時に RSA キーペアの秘密鍵を利用する
* 公開鍵証明書
* ...

いろんな用語、トピックがあり混乱するのでまとめてみた。

ここではとりあえず特にRSA暗号を念頭に置き話を進める。

## 規格群

そもそも公開鍵暗号に関する技術はいろいろな形で規格が定められていたり、標準化されていたりする。

### PKCS
Public-Key Cryptography Standards の略で、RSAセキュリティというソフトウェア会社が考案した公開鍵暗号の規格群のこと。元々はRSAセキュリティ社が自社の暗号技術に関する特許を利用促進するために発行したのがはじまり。
近年ではその一部は IETF などと標準化が進められており、 RFC として整備されているものも多い。

内容ごとに PKCS #1 など番号が振られている。詳細は後述。
https://en.wikipedia.org/wiki/PKCS


### ITU-T / Xシリーズ勧告

ITU-T は国際連合の専門機関で、世界規模で電気通信に関する技術を標準化することが目的。
"X.509" などのようにアルファベットと数字の組み合わせで、国際標準となる勧告を整理しており、 ITU-T勧告と呼ばれる。 "X.nnn" の形式の勧告はXシリーズ勧告と呼ばれ、「データ網及びオープン・システム・コミュニケーション」に関する規定を定めている。
例えば、

* X.509 は公開鍵証明書の標準形式を定めたもの
    * [RFC 5280]( https://datatracker.ietf.org/doc/html/rfc5280 )
* X.680シリーズは ASN.1 について定めている
* X.690 は ASN.1 のエンコーディング形式について定めたもの
    * いわゆる DER 形式のエンコーディングはここで定められている
    * https://en.wikipedia.org/wiki/X.690


### RFC

Request for Comments の略で IETF (Internet Engineering Task Force) によるインターネット技術の標準的な仕様の文書のこと。

https://ja.wikipedia.org/wiki/Request_for_Comments



## RSA暗号

RSA暗号は、巨大な自然数の素因数分解が一般に困難（高速にはできない）ことを利用した暗号技術。

<!-- markdown-link-check-disable-next-line --> 
![rsa-explanation](/images/articles/private-key-summary/rsa-encryption.png)
*[slideshare - RSA暗号運用でやってはいけない n のこと #ssmjp]( https://www.slideshare.net/sonickun/rsa-n-ssmjp ) より引用*


本質的には二つの大きな素数 p と q が秘密鍵で、その積 n と暗号化指数 e （65537が一般的）が公開鍵。実際には計算を高速化するためにもう少し他の数字も鍵に入っていたりする。後述の PKCS #1 で記載されている。

RSA暗号自体の詳細は別の記事を適宜参照してください。

## ASN.1

Abstract Syntax Notation One（抽象構文表記 1） の略で、データ構造などを表現する柔軟な記法のこと。ASN.1 自体は公開鍵暗号のデータ構造以外にも使える一般的なもの。
データ構造はもちろん特定のプログラミング言語でも定義できるが、 ASN.1 で表記することで言語に依存しない形で表現できるのが嬉しい。

https://ja.wikipedia.org/wiki/Abstract_Syntax_Notation_One / https://en.wikipedia.org/wiki/ASN.1

## PKCS
内容ごとに数字が割り振られている。

### PKCS #1

RSA Cryptography Standard で、がこれに該当。VersionごとにRFCがある。
* [RFC 8017]( https://datatracker.ietf.org/doc/html/rfc8017 ) : PKCS #1 Version 2.2
* [RFC 2313]( https://datatracker.ietf.org/doc/html/rfc2313 ) : PKCS #1 Version 1.5

RSA暗号の実装に関する推奨や、RSA暗号の公開鍵や秘密鍵の ASN.1 表記でのデータ構造が記載されている。

```asn1
 RSAPrivateKey ::= SEQUENCE {
     version           Version,
     modulus           INTEGER,  -- n
     publicExponent    INTEGER,  -- e
     privateExponent   INTEGER,  -- d
     prime1            INTEGER,  -- p
     prime2            INTEGER,  -- q
     exponent1         INTEGER,  -- d mod (p-1)
     exponent2         INTEGER,  -- d mod (q-1)
     coefficient       INTEGER,  -- (inverse of q) mod p
     otherPrimeInfos   OtherPrimeInfos OPTIONAL
 }
```

`openssl genrsa` でデフォルトで生成される鍵はこの PKCS #1 に従ったもの。

### PKCS #8

Private-Key Information Syntax Standard で、[RFC 5958]( https://datatracker.ietf.org/doc/html/rfc5958 ) がこれに該当。
RSA暗号に限らず、秘密鍵のフォーマットについて記載されている。
`openssl pkcs8` で扱える。

## Encoding

ASN.1 表記のデータ構造をコンピューター間でやりとりするにはエンコードしてバイト列にする必要がある。そのやり方はいくつかある。

### BER
ASN.1 のデータ構造を Type-Length-Value をペアにしてエンコードしていく方式。

詳しくは以下を参照
* https://en.wikipedia.org/wiki/X.690


### DER
Distinguished Encoding Rules の略。BER ( Basic Encoding Rules ) の一種で、一部正規化されたもの。

詳しくは以下を参照
* [Let's Encrypt - ASN.1 と DER へようこそ]( https://letsencrypt.org/ja/docs/a-warm-welcome-to-asn1-and-der/ )
* [RSA暗号とPEM/DERの構造]( https://www.sambaiz.net/article/135/ )


### PEM
Privacy-Enhanced Mail の略。DER は ASN.1 のデータ構造をバイト列としてエンコーディングするが、このままだとやり取りする際に扱いにくい。
そこで、 DER でエンコードされたバイト列を Base64 でエンコーディングしてテキストにし、ヘッダー（ `-----BEGIN{label}-----` ）とフッター（`-----END{label}-----`）で囲んだものが PEM 形式のキー。ラベルがエンコーディングされているメッセージが何かを表している。

このヘッダー・フッターや、 Base64 でエンコードされた鍵そのものは64文字ごとに改行する、といったルールは、 [RFC 7468 - Textual Encodings of PKIX, PKCS, and CMS Structures]( https://tex2e.github.io/rfc-translater/html/rfc7468.html ) で定義されている。


https://en.wikipedia.org/wiki/Privacy-Enhanced_Mail


## ファイル拡張子

拡張子はいろいろありえる（？）

* エンコーディングを明示するパターン
    * `.der` や `.pem` など
    * この場合、ファイルの中身は秘密鍵のこともあれば証明書のこともある
* ファイルの中身を明示するパターン
    * `.crt` や `.cer` また `.key` など
    * この場合、そのファイルのエンコーディングは DER のこともあれば PEM のこともある
* データ構造を明示するパターン
    * `.p8` や `.p12` など
    * PKCS のどれに従っているかを明示している


## OpenSSL

### PKCS#1 の秘密鍵を PEM で出力する
openssl はデフォルトでは PKCS #1 に従ったデータ構造で、PEM でエンコードする。

```console
$ openssl genrsa 2048 > test_p.key 
Generating RSA private key, 2048 bit long modulus
..............................+++
...............+++
e is 65537 (0x10001)
$ cat test_p.key
-----BEGIN RSA PRIVATE KEY-----
MIIEowIBAAKCAQEAo338PxtQqYCbD3nGTE+W9CjqN3RS2J94xoHpyAY7yRBBhqem
0HuPwGmYeQJksc4uTFmBomOzhLj8MzAb4rtu/IMXKVE0//AB5iKoqubgSy1CZEOH
...
kkyxB8jhKTN42fQqWF8q2HuI+LNbYomrP6dd8vD5rPASOHOHx6lT
-----END RSA PRIVATE KEY-----
```

### 秘密鍵の中身を詳細に出力する
`openssl rsa` で `-text` オプションを使うと表示できる。

```console
$ cat test_p.key | openssl rsa -noout -text
Private-Key: (2048 bit)
modulus:
    00:a3:7d:fc:3f:1b:50:a9:80:9b:0f:79:c6:4c:4f:
    ...
publicExponent: 65537 (0x10001)
privateExponent:
    26:28:81:77:39:28:da:66:e9:c9:f2:e2:15:6d:7e:
    ...
prime1:
    00:d2:c3:da:c8:a3:27:6b:bd:91:e2:03:98:8f:9e:
    ...
prime2:
    00:c6:94:cb:d1:14:57:3f:d4:93:07:e5:4b:a3:44:
    ...
exponent1:
    59:6e:01:47:60:f3:39:24:16:e2:6f:e4:2c:0c:a1:
    ...
exponent2:
    4f:4b:31:0b:7e:9c:cc:3f:1c:aa:c5:73:6b:71:28:
    ...
coefficient:
    00:c1:62:f5:78:50:dd:fb:a9:57:9f:eb:40:ee:90:
    ...
```

実際に PKCS #1 で定義されている情報が入っていることがわかる。

### ASN.1 のデータとしてパースする
`openssl asn1parse` を使えばパースした結果をそのまま見ることもできる。
`openssl rsa -noout -text` と同じ情報が入っていることが確認できる。
```console
$ cat test_p.key | openssl asn1parse -inform PEM
    0:d=0  hl=4 l=1187 cons: SEQUENCE
    4:d=1  hl=2 l=   1 prim: INTEGER           :00
    7:d=1  hl=4 l= 257 prim: INTEGER           :A37DFC3F1B50A9809B...
  268:d=1  hl=2 l=   3 prim: INTEGER           :010001
  273:d=1  hl=4 l= 256 prim: INTEGER           :262881773928DA66E9...
  533:d=1  hl=3 l= 129 prim: INTEGER           :D2C3DAC8A3276BBD91...
  665:d=1  hl=3 l= 129 prim: INTEGER           :C694CBD114573FD493...
  797:d=1  hl=3 l= 128 prim: INTEGER           :596E014760F3392416...
  928:d=1  hl=3 l= 128 prim: INTEGER           :4F4B310B7E9CCC3F1C...
 1059:d=1  hl=3 l= 129 prim: INTEGER           :C162F57850DDFBA957...
```

### PKCS #8 の秘密鍵を生成する
`openssl pkcs8` を使うと変換できる。
```console
$ cat test_p.key | openssl pkcs8 -topk8 -inform PEM -nocrypt
-----BEGIN PRIVATE KEY-----
MIIEvQIBADANBgkqhkiG9w0BAQEFAASCBKcwggSjAgEAAoIBAQCjffw/G1CpgJsP
ecZMT5b0KOo3dFLYn3jGgenIBjvJEEGGp6bQe4/AaZh5AmSxzi5MWYGiY7OEuPwz
...
ias/p13y8Pms8BI4c4fHqVM=
-----END PRIVATE KEY-----
```

`open genrsa` で生成された PEM は `BEGIN RSA PRIVATE KEY` だったのが、 `openssl pkcs8` を使うことで `BEGIN PRIVATE KEY` になり中身も変わっていることがわかる。

### その他の変換

openssl のコマンドをうまく使えば大抵なんでもできる。以下の記事などを参照。

https://qiita.com/ling350181/items/2ac60698779088b14dea

https://qiita.com/kunichiko/items/12cbccaadcbf41c72735



### openssl docs

* https://www.openssl.org/docs/man1.1.1/man1/genrsa.html
* https://www.openssl.org/docs/man1.1.1/man1/rsa.html
* https://www.openssl.org/docs/man1.1.1/man1/openssl-pkcs8.html

## References

### 記事

https://letsencrypt.org/ja/docs/a-warm-welcome-to-asn1-and-der/
https://www.oresamalabo.net/entry/2020/03/01/194300
https://stackoverflow.com/questions/10783366/how-to-generate-pkcs1-rsa-keys-in-pem-format
https://www.tohoho-web.com/ex/openssl.html
https://www.sambaiz.net/article/135/
https://zenn.dev/htlsne/articles/rsa-key-format


### RFC

https://datatracker.ietf.org/doc/html/rfc5280
https://datatracker.ietf.org/doc/html/rfc5958
https://datatracker.ietf.org/doc/html/rfc7468
https://datatracker.ietf.org/doc/html/rfc8017