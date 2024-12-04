---
title: "Snowflake の外部テーブル入門"
emoji: "❄️"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Snowflake", "DataEngineering", "SQL", "dbt"]
published: true
publication_name: dataheroes
---

## はじめに

こんにちは！ナウキャストのデータエンジニアのけびんです。

先日 Snowflake Unconference #2 に参加し、外部テーブルの話をしました。この記事はその際の資料として利用したものです。
https://techplay.jp/event/949829

---

皆さん Snowflake へデータを取り込むときにどのような方法を使っているでしょうか？
Snowpipe を利用するのが王道パターンかと思いますが、僕は外部テーブルを利用する方法が好きです。

外部テーブルの存在は知っていても細かいところが意外と知られていないと感じることが多いので、本記事ではその辺りを紹介していこうと思います。


## 外部テーブルとは

外部テーブルとは、顧客が管理するS3などの外部ステージに格納されているデータに対して、それらがあたかも Snowflake のテーブル内にあるかのようにクエリがすることができる機能です。

AWS で Athena を利用したことがある方は、 Athena のテーブルを想像していただくと分かりやすいと思います。


以下のようなポイントがあります。

* 外部テーブルは読み取り専用
* パフォーマンスについて
  * `CREATE EXTERNAL TABLE` については基本的に早い印象
  * 外部テーブルに対するクエリはネイティブなテーブルに比べて基本的にパフォーマンスが悪い
    * S3などに毎回データを取りに行く必要があるため
* クローンはできない
* `metadata$filename` という特殊なカラムがある
* ファイルサイズの推奨事項
  * 並列処理をいい感じにやるために、Parquetであれば256~512MBのサイズのファイルにすることが推奨されている
* パーティションの設定が可能
  * `metadata$filename` がここでも活躍します
* dbt から外部テーブルを作成することも可能


## 外部テーブルを実際に使ってみる

### サンプルデータの用意

サクッと試すだけなので特に意味はないですが３つファイルを用意します。
これらのファイルを `s3://<bucket-name>/db1/table1/hoge.csv` という場所に配置しておきます。

```csv:data_20240701.csv
col1,col2
A,B
C,D
```

```csv:data_20240702.csv
col1,col2
AAA,BBB
CCC,DDD
EEE,FFF
```

```csv:data_20240703.csv
col1,col2
W,W
X,X
Y,Y
Z,Z
```


### ストレージインテグレーションの用意

外部テーブルは外部ステージにあるデータを取り扱うわけですが、外部ステージつまり自分たちが管理しているS3などへ Snowflake から安全にアクセスする設定が必要です。このためにはストレージインテグレーションの設定が必要になります。
AWS S3 を念頭に概要を説明すると、 Snowflake 側の AWS アカウント内の IAM User が、自分の管理する AWS アカウント内の IAM Role へとクロスアカウントで Assume Role するための諸々の設定をする、ということになります。

このあたりも奥が深いのですが、以下のドキュメントなどを参考に用意ができていると仮定します。
https://docs.snowflake.com/ja/user-guide/data-load-s3-config-storage-integration

AWS の IAM やクロスアカウントアクセスに関しては以下の自分のスライドも参考になるかもしれません。
https://speakerdeck.com/kevinrobot34/introduction-aws-iam-3a810adc-5172-4460-9721-0041456eb2bf?slide=32


* Snowflake 側での作業
  ```sql
  CREATE OR REPLACE STORAGE INTEGRATION storage_integration_tutorial
  TYPE = EXTERNAL_STAGE
  STORAGE_PROVIDER = 'S3'
  ENABLED = TRUE
  STORAGE_AWS_ROLE_ARN = 'arn:aws:iam::<account-id>:role/snowflake-external-table-tutorial'
  STORAGE_ALLOWED_LOCATIONS = ('s3://<bucket-name>/');
  ```
  * `DESC INTEGRATION storage_integration_tutorial;` で `STORAGE_AWS_IAM_USER_ARN` と `STORAGE_AWS_EXTERNAL_ID` を控えておく
* AWS 側での設定
  * S3バケットを作成（ `STORAGE_ALLOWED_LOCATIONS` で指定した名前で作る ）
    * サンプルデータとして用意したファイルたちを配置しておく
  * IAM Role の作成 ( `STORAGE_AWS_ROLE_ARN` で指定した名前で作る )
    * Trust relationship に先ほど控えた `STORAGE_AWS_IAM_USER_ARN` と `STORAGE_AWS_EXTERNAL_ID` の情報を含める


### 外部ステージの作成

ファイルフォーマットと外部ステージも用意しておきましょう
```sql
CREATE OR REPLACE FILE FORMAT my_csv_format
  TYPE = 'CSV'
  FIELD_DELIMITER = ','
  SKIP_HEADER = 1;

CREATE OR REPLACE STAGE my_s3_stage
  STORAGE_INTEGRATION = storage_integration_tutorial
  URL = 's3://<bucket-name>/db1/table1/'
  FILE_FORMAT = my_csv_format;
```

`list @my_s3_stage;` で以下のようにファイルの一覧が表示されたらOKです！
![stage_list_res](/images/articles/snowflake-external-table/stage_list_res.png)


### パーティションカラムの用意

外部テーブルにはパーティションを設定することができます。
プルーニングするためには必須なので設定しましょう。

外部ステージには `metadata$filename` という特殊なカラムがありこれを利用します。
https://docs.snowflake.com/en/user-guide/querying-metadata

https://docs.snowflake.com/en/user-guide/tables-external-intro#schema-on-read

```sql
select 
    metadata$filename, 
    metadata$file_row_number,
from @my_s3_stage/
limit 100;
```

![stage_filename](/images/articles/snowflake-external-table/stage_filename.png)


今回の場合 `db1/table1/data_20240701.csv` のうちファイル名の `20240701` の部分を `YYYYMMDD` の日付のパーティションカラムとして使うようにしたいです。

`split_part` や `substr` などを使い必要な部分を抽出すれば良いです。これが結構面倒なので、以下のようなクエリを Worksheet でトライアンドエラーしながら綺麗にfilenameをパースできているかを確認するのがおすすめです。

![stage_partition_try_and_error](/images/articles/snowflake-external-table/stage_partition_try_and_error.png)

最終的には日付の部分を抜き出すには以下のようにパースすれば良いです。
```sql
TO_DATE(
    split_part(split_part(split_part(metadata$filename, '/', 3), '_', 2), '.', 1), 
    'YYYYMMDD'
)
```

### 外部テーブルの作成

```sql
create or replace external table sample_et (
    col1 varchar as (value:c1::varchar),
    col2 varchar as (value:c2::varchar),
    date_part date as TO_DATE(
        split_part(split_part(split_part(metadata$filename, '/', 3), '_', 2), '.', 1), 
        'YYYYMMDD'
    )
)
PARTITION BY (date_part)
LOCATION=@my_s3_stage
AUTO_REFRESH = false
FILE_FORMAT = my_csv_format
;
```

適切にパーティション切っていれば、そのカラムを使うとプルーニングされて早くなるので必須です！



## dbt で外部テーブルを利用する

`dbt-external-tables` というパッケージを利用することで、 dbt で外部テーブルを定義することが簡単にできるようになります。
https://github.com/dbt-labs/dbt-external-tables

使い方としては、 dbt source として通常通りテーブルの情報を記載するのですが、 `external` というセクションを追加し、
* 外部テーブルのロケーション
* ファイルフォーマット
* パーティション

といった外部テーブルに関連する追加情報を記載します。

```yaml
version: 2

sources:
  - name: snowplow
    database: analytics
    schema: snowplow_external
    loader: S3
    loaded_at_field: collector_hour
    
    tables:
      - name: event_ext_tbl
        description: "External table of Snowplow events stored as JSON files"
        external:
          location: "@raw.snowplow.snowplow"  # reference an existing external stage
          file_format: "( type = json )"      # fully specified here, or reference an existing file format
          auto_refresh: true                  # requires configuring an event notification from Amazon S3 or Azure
          partitions:
            - name: collector_hour
              data_type: timestamp
              expression: to_timestamp(substr(metadata$filename, 8, 13), 'YYYY/MM/DD/HH24')
              
        # all Snowflake external tables natively include a `metadata$filename` pseudocolumn
        # and a `value` column (JSON blob-ified version of file contents), so there is no need to specify
        # them here. you may optionally specify columns to unnest or parse from the file:
        columns:
          - name: app_id
            data_type: varchar(255)
            description: "Application ID"
          - ...
```
* https://github.com/dbt-labs/dbt-external-tables/blob/main/sample_sources/snowflake.yml より参照


これらを踏まえ、 `dbt run-operation stage_external_sources` とコマンドを実行すると、適宜この dbt source の情報を踏まえた `CREATE EXTERNAL TABLE` 文が実行されるというようなイメージです。
このように source として外部テーブルを作成したら、あとは普通に dbt でパイプラインを作っていけばOKです。


## 外部テーブルでのデータのロード

Snowflake へデータをロードするために外部テーブルを使うことが可能なわけですが、メリット・デメリットがあります。

* メリット
  * Athena などを利用していて S3 に綺麗にデータが入っている場合サクッとSnowflakeにデータをロードできる
  * 元データを直接DWHから簡単に調査可能
  * dbt を組み合わせることで dbt のコマンドでデータの取り込みができたり、複雑なインクリメンタルな取り込みもできる
    * パイプラインをすべて dbt の世界に寄せられる
* デメリット
  * 外部テーブルはパフォーマンスの問題がある
    * パイプラインに外部テーブルをネイティブなテーブルに変換するステップを入れておく必要があり、また差分更新しないと厳しかったりする
      * dbt であれば incremental model との組み合わせが実質必須
    * もしくはマテリアライズドビューを使ったり
  * 外部テーブルはクローンできない

この辺りに気をつけた上で使うとかなり有効な機能かなと思います。


## 注意事項

### 外部テーブルのスキーマ

https://docs.snowflake.com/ja/user-guide/tables-external-intro#schema-on-read で記載がある通り、外部テーブルには以下の３つの特殊な列がある。
* `VALUE`
  * 外部ファイルのある行のレコードを表す VARIANT 型の列
  * 外部テーブルに対する `SELECT *` は必ず VALUE 列を含む
  * 外部ステージに配置されたファイルの形式によって `VALUE` に含まれる列の値も変わる
    * PARQUETの場合、カラム名が key となっている `VARIANT`
    * CSVの場合、 `c1` などのような値が key となっている `VARIANT`
* `METADATA$FILENAME`
  * 外部ステージに配置されたデータファイルの名前
  * S3の場合はオブジェクトの key が対応する
* `METADATA$FILE_ROW_NUMBER`
  * 外部ステージに配置されたデータファイルの各レコードの行番号


### `dbt-external-tables` の注意

* 外部ステージに配置したファイルが parquet の場合、 source の定義においてカラム名は大文字で定義しておいた方が良い
  * 生成されるddlは以下のような形式で、 `value:row_id` として参照するが、 value 列では key が大文字になったりしていてうまく参照できなかったりすることがある。
    ```sql
    create or replace external table database.schema.test_external_table(
      row_id varchar(255) as ((case when is_null_value(value:row_id) or lower(value:row_id) = 'null' then null else value:row_id end)::varchar(255)),
    ...
    ) partition by (by_scraping_date) 
    location = @schema.sample_stage/test/path 
    file_format = ( TYPE = PARQUET, COMPRESSION = SNAPPY, BINARY_AS_TEXT = FALSE)
    ```


## まとめ

本ブログでは Snowflake の外部テーブルについて紹介してきました。Partition を使い dbt と組み合わせることで柔軟に管理・運用が可能です。

S3に比較的綺麗にデータが揃っている場合などにはかなり有効なデータのロード方法だと思っています。機会があればぜひ使ってみてください！