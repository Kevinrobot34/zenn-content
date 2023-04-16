---
title: "Pythonで非同期処理をやる"
emoji: "🚥"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Python", "非同期処理", "asyncio"]
published: true
---

# 用語

## ブロッキング・ノンブロッキング

* ブロッキング
    * プログラムのある処理が完了するのを待っており、他の処理は行えない状態のこと
    * CPU等のリソースを占有してしまうが、入出力などの場合にはレスポンスをただ待っているだけということがあり、勿体無い
* ノンブロッキング
    * プログラムのある処理の待機時に、ブロッキングせず途中で他の処理に移ることが可能な状態のこと
    * 適切にノンブロッキングなコードを書き、リソースの使用率を高めたり無駄な待機時間をなくしたりすることで、処理の高速化を狙う


## 並行処理と並列処理

似た単語だけど意味するところは結構違うので注意

* 並行処理 (Concurrency)
    * プログラムを複数の独立に実行できる子タスクとして設計し、有限の計算リソースをうまく使い近似的に同時処理をする感じ
    * 入出力の待ち時間でCPUが計算をする必要がない間には他のタスクを割り込ませるみたいな
* 並列処理 (Parallelism)
    * 複数のタスクを同時に処理すること
    * 余分な計算リソースを利用し、ある時刻において複数のタスクを同時に実行する
* 表にするとこんな違いがある
    | 観点 | 並列処理 | 並行処理 |
    |:---:|:---:|:---:|
    | 英語 | Parallelism | Concurrency |
    | 時間 | ある**時点**で複数のタスクが処理されている | ある**期間**で複数のタスクが処理される<br>（ある時点で複数のタスクが処理されてない） |
    | 対象 | 複数の処理を同時に**実行する**<br>（プログラムの実行の仕方の話） | 複数の処理を独立に実行できる**構成**<br>（コードの構成の話） |
    | レイヤー | ハードウェアの特性 | プログラミングバターン |


## イベントループ

非同期処理を実装するためのデザインパターンの一つ。
asyncioの高レベルAPIを叩いてる分にはそんなに意識しないで良いが、知っといた方がしっくりくるかも。
https://docs.python.org/ja/3/library/asyncio-eventloop.html#asyncio-event-loop


# asyncio

I/O（入出力）の待ち時間に別の処理をやらせることで **並行処理** をさせるための仕組み。

入出力の処理は特にネットワーク通信が入ると、どれくらい時間がかかるかわからず、その間ただCPUに何もせず待たせている（ブロッキングする）のは勿体無い。そこで、「あるIO処理の終了を待たずに他の処理をする」ことで、近似的に同時にタスクを処理し、プログラム全体にかかる時間を減らそう、という感じ。

> threadingやmultiprocessでも似たように、処理の終了を待たずに他の処理をすることができるが、asyncioによる非同期IO処理はそれよりももっと軽量に動作し、実際には並列して処理が動くことはなく1スレッドで動いている処理は1つだけになる。
> 
> 「同時に処理が動かない」のに「処理の終了を待たずに他の処理ができる」のは奇妙に思うかもしれない。IO処理というのはCPUを使わず処理が終わるのを待っている時間がある。例えばファイルの読み込みをしている間は、CPUを使わずにファイルのデータが読み込まれるのを待っている。この時間はCPU的には十分に長い時間なのでこの時間に他のことが処理できる。
> 
> 非同期IO処理は「空き時間にこの処理をして！終わったら教えて！」という細かい処理をどんどん登録して、「空いた！」「じゃあこれやって！」という小さな処理単位をイベントループというループの中でどんどん処理する。処理自体は同時に1つしかこなせないが、空き時間を有効活用していることになる。ちょうど料理中に電子レンジをONにして、チンと完了するまでに他の作業をするのに似た感じ。
> [asyncio 非同期IOの基本]( https://python.civic-apps.com/asyncio-basic/ )


## api ref

### async

コルーチンを定義するのが `async def` 文。

### await

`await <coroutine>` な感じで使う。
`await` をつけることで、そのコルーチンはI/Oなどの待機時間が発生してしまうのでその間に別の処理を割り込ませて良い、ということを明示する感じ。


### asyncio.run

> `asyncio.run(coro, *, debug=None)`
> coroutine coro を実行し、結果を返します。
> この関数は、非同期イベントループの管理と 非同期ジェネレータの終了処理 およびスレッドプールのクローズ処理を行いながら、渡されたコルーチンを実行します。
> https://docs.python.org/ja/3/library/asyncio-runner.html#asyncio.run

非同期IOのエントリーポイントとして使うイメージ。


### asyncio.gather

> `awaitable asyncio.gather(*aws, return_exceptions=False)`
> aws シーケンスにある awaitable オブジェクト を 並行 実行します。
> aws にある awaitable がコルーチンである場合、自動的に Task としてスケジュールされます。
> 
> 全ての awaitable が正常終了した場合、その結果は返り値を集めたリストになります。 返り値の順序は、 aws での awaitable の順序に相当します。
> https://docs.python.org/ja/3/library/asyncio-task.html#asyncio.gather

並行にコルーチンをタスクとして実行してくれる。


## 具体例1: time

### gatherを使って並行実行
```python
import asyncio
import time

async def say_after(delay, what):
    await asyncio.sleep(delay)
    print(f"{time.strftime('%X')} - {what}")

async def main():
    print(f"{time.strftime('%X')} - start")
    c1 = say_after(1, 'hello')
    c2 = say_after(2, 'world')
    await asyncio.gather(c1, c2) #2つの処理の終了を待つ
    print(f"{time.strftime('%X')} - finish")


# 非同期IOのエントリポイント
asyncio.run(main())
# 13:50:10 - start
# 13:50:11 - hello
# 13:50:12 - world
# 13:50:12 - finish
```

### 適当にやると同期処理と変わらん例１
```python
import asyncio
import time

async def say_after(delay, what):
    await asyncio.sleep(delay)
    print(f"{time.strftime('%X')} - {what}")

async def main():
    print(f"{time.strftime('%X')} - start")
    await say_after(1, 'hello')
    await say_after(2, 'world')
    print(f"{time.strftime('%X')} - finish")


asyncio.run(main())

# 13:50:33 - start
# 13:50:34 - hello
# 13:50:36 - world
# 13:50:36 - finish
```


### 適当にやると同期処理と変わらん例２
```python
import asyncio
import time

async def say_after(delay, what):
    time.sleep(delay)  # asyncio.sleep ではなく、 time.sleep
    print(f"{time.strftime('%X')} - {what}")

async def main():
    print(f"{time.strftime('%X')} - start")
    c1 = say_after(1, 'hello')
    c2 = say_after(2, 'world')
    await asyncio.gather(c1, c2) #2つの処理の終了を待つ
    print(f"{time.strftime('%X')} - finish")

asyncio.run(main())
# 14:12:41 - start
# 14:12:42 - hello
# 14:12:44 - world
# 14:12:44 - finish
```

非同期処理を書く際の注意点として、コルーチン（async defの関数）の中でブロッキング処理をしてしまうと意味ないというのがある。
上記のようにブロッキング処理である `time.sleep` を使うと、ブロックされてしまうのでこの間に他の処理を走らせるということができなくなってしまう。


## 具体例2: api call

OpenAIのAPIなどはうらで大規模言語モデルが動いているのでAPIのレスポンスが帰ってくるまでに数秒単位で時間を要することはよくある。

このようなAPIに対して asyncio を使うのは非常に効果的。

以下の例では、openaiのドキュメントの文を日本語に翻訳するようにAPIを叩いているが、asyncioをうまく使うと並行処理により高速化されていることがわかる。

```python
import asyncio
import time

import openai

texts = [
    "The OpenAI API can be applied to virtually any task that involves understanding or generating natural language, code, or images.",
    "We offer a spectrum of models with different levels of power suitable for different tasks, as well as the ability to fine-tune your own custom models. ",
    "These models can be used for everything from content generation to semantic search and classification.",
    "We recommend completing our quickstart tutorial to get acquainted with key concepts through a hands-on, interactive example.",
]


def throw_query_sync(text: str) -> str:
    response = openai.ChatCompletion.create(
        model="gpt-3.5-turbo",
        messages=[
            {"role": "user", "content": f"Translate the following English text to Japanese: \"{text}\""},
        ],
    )
    return response.choices[0]["message"]["content"].strip()

def main_sync():
    start_time = time.time()
    results = [throw_query_sync(text) for text in texts]
    end_time = time.time()
    overall_time = end_time - start_time
    print(f"Overall time: {overall_time} (average {overall_time/len(texts)})")
    print(*results, sep='\n')

async def throw_query_async(text: str) -> str:
    response = await openai.ChatCompletion.acreate(
        model="gpt-3.5-turbo",
        messages=[
            {"role": "user", "content": f"Translate the following English text to Japanese: \"{text}\""},
        ],
    )
    return response.choices[0]["message"]["content"].strip()

async def main_async():
    start_time = time.time()
    tasks = [throw_query_async(text) for text in texts]
    results = await asyncio.gather(*tasks)
    end_time = time.time()
    overall_time = end_time - start_time
    print(f"Overall time: {overall_time}")
    print(*results, sep='\n')


main_sync()
asyncio.run(main_async())
```

```
Overall time: 11.700689315795898 (average 2.9251723289489746)
OpenAIのAPIは、自然言語、コード、または画像の理解や生成に必要なほとんどのタスクに適用できます。
私たちは、異なるタスクに適した異なるパワーレベルを持つモデルのスペクトラムを提供し、カスタムモデルを微調整する能力も提供しています。
これらのモデルは、コンテンツ生成から意味検索や分類に至るまで、あらゆる用途に使えます。
わたしたちは、キーポイントを実践的でインタラクティブな例を通して理解するため、クイックスタートチュートリアルの完了をお勧めします。
Overall time: 4.258472204208374
OpenAI APIは、自然言語、コード、または画像の理解や生成を必要とするほとんどのタスクに適用することができます。
私たちは、異なるタスクに適した様々なパワーレベルを持つモデルのスペクトルを提供しています。さらに、独自のカスタムモデルを微調整する能力も持っています。
これらのモデルは、コンテンツの生成から意味的な検索や分類にまで使用できます。
当社のクイックスタートチュートリアルを完了することをお勧めします。こちらでは、実践的でインタラクティブな例を通じて、キーコンセプトについて馴染みを深めることができます。
```


# threading や multiprocess

* [Qiita - Pythonの非同期プログラミングを完全理解]( https://qiita.com/kaitolucifer/items/3476158ba5bd8751e022 ) に具体例が結構かいてある
    * `concurrent.futures.ProcessPoolExecutor` を用いた[例]( https://qiita.com/kaitolucifer/items/3476158ba5bd8751e022#3-2-%E5%90%8C%E6%9C%9F%E3%83%96%E3%83%AD%E3%83%83%E3%82%AD%E3%83%B3%E3%82%B0%E6%96%B9%E5%BC%8F%E3%81%AE%E6%94%B9%E8%89%AF%E3%83%9E%E3%83%AB%E3%83%81%E3%83%97%E3%83%AD%E3%82%BB%E3%82%B9%E6%96%B9%E5%BC%8F )
    * `concurrent.futures.ThreadPoolExecutor` を用いた[例]( https://qiita.com/kaitolucifer/items/3476158ba5bd8751e022#3-3-%E5%90%8C%E6%9C%9F%E3%83%96%E3%83%AD%E3%83%83%E3%82%AD%E3%83%B3%E3%82%B0%E6%96%B9%E5%BC%8F%E3%81%AE%E6%9B%B4%E3%81%AA%E3%82%8B%E6%94%B9%E8%89%AF%E3%83%9E%E3%83%AB%E3%83%81%E3%82%B9%E3%83%AC%E3%83%83%E3%83%89%E6%96%B9%E5%BC%8F )



# References

## 公式ドキュメント

https://docs.python.org/ja/3/library/asyncio.html
https://docs.python.org/ja/3/library/asyncio-eventloop.html


## blogs

https://iuk.hateblo.jp/entry/2017/01/27/173449
https://qiita.com/kaitolucifer/items/3476158ba5bd8751e022
https://qiita.com/everylittle/items/57da997d9e0507050085
https://python.civic-apps.com/asyncio-basic/

## その他

https://fastapi.tiangolo.com/async/
https://blog.ojisan.io/server-architecture-2023/
https://qiita.com/kaitolucifer/items/e4ace07bd8e112388c75
https://zenn.dev/hsaki/books/golang-concurrency/viewer/term