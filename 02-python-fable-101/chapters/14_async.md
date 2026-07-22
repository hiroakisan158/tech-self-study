# 第14章 行列をさばく — 非同期処理と並行性

## 🏪 今日のお話

お店は大繁盛。しかし問題が発生しました。

エリクサーの注文が入ると、店主は **配達妖精が材料を取ってくるまでの 3 秒間、
何もせずに待ちます**。その間、後ろの行列はイライラ…。

よく観察すると、店主は「働いている」のではなく「**待っている**」だけ。
待ち時間に次のお客さんの注文を受ければいいのです。
これが **非同期処理(async)** の発想です。

### 実務ではどんな処理で使われるのか

「妖精を待つ」は比喩ですが、実務でも **「待ち時間はあるのに、その間 CPU は何もしていない」処理** は
至るところにあります。共通するのは「相手(ネットワークの向こう・DB・OS)の応答を待つだけで、
自分では計算していない時間がある」ことです。代表例:

- **Web API サーバー**: 100 人が同時にアクセスしてきても、各リクエストが DB の応答を待っている間に
  他のリクエストを処理できる(FastAPI などが典型)。同期式だと 1 人の応答待ちが他の全員を止めてしまう
- **HTTP クライアント / クローラー**: 100 個の URL から情報を集めたいとき、1 件ずつ順番に待つと
  「通信時間 × 100」かかるが、`asyncio.gather` で同時に投げれば「一番遅い1件分」で済む
  (この章のはじめの例と全く同じ構造)
- **チャットボット / Discord・Slack bot**: 誰かのメッセージを待ちながら、同時に別ユーザーの
  メッセージやタイマーイベントにも反応する必要がある常駐プログラム
- **DB へのバッチ問い合わせ**: 複数のクエリを同時に投げて、応答をまとめて待つ(1 件ずつ順番に
  待つより大幅に速い)
- **複数の外部サービスへの問い合わせ集約(fan-out)**: 1 つのリクエストに応えるために、
  在庫サービス・料金サービス・配送サービスなど複数の外部 API に同時に問い合わせ、
  全部揃ってから結果をまとめて返す

逆に言えば、**待ち時間がほとんどない処理(画像処理・数値計算など)には async は向きません**。
これは後述する「I/O バウンド vs CPU バウンド」の話に直結します。
この章末の完成コード(`asyncio.Queue` + ワーカー)は、まさにこの「クローラー・チャットボット・
API サーバー」で最頻出のパターンです。

### 大前提: 非同期処理は「並列処理」ではない

ここが一番の勘違いポイントです。店主は **1 人のまま** です。増員(スレッドやプロセスを増やす)は
していません。店主が同時に手を動かしている瞬間は一度もなく、やっているのは

1. エリクサーの注文を受け、妖精を送り出す(ここまでは店主の仕事)
2. 妖精が戻るまでの間、**手が空くので** 次の回復薬の注文を受け、妖精を送り出す
3. また手が空くので、マナポーションの注文を受け、妖精を送り出す
4. 3 人分の妖精が(ほぼ同時に)戻ってきたら、順番に商品を渡す

という「**手が空いた瞬間に次の仕事に切り替える**」を高速に繰り返しているだけです。
実際に手を動かしているのはいつも店主 1 人 ―― つまり **同時に処理が進んでいるように見えるだけで、
実行の主体は常に1つ**。これを「並列(parallel)」と区別して **並行(concurrency)** と呼びます。

- **並列**: 店員が 3 人いて、3 人が同時に接客する(= マルチスレッド/マルチプロセス)
- **並行**: 店員は 1 人のまま、待ち時間の間だけ他の注文をこまめに拾う(= 非同期処理)

非同期処理が速くなるのは「店主がサボらず手を動かし続けるから」であって、
「店主が増えるから」ではありません。この違いを頭に入れてから先を読むと迷いにくくなります。

## 同期の世界 — 1 人ずつ順番に

```python
import time

def fetch_ingredient(name: str) -> str:
    print(f"  🧚 {name} を取りに行きます…")
    time.sleep(3)                      # 妖精の往復(この間、店は完全停止)
    return f"{name} の材料"

def serve_three() -> None:
    start = time.perf_counter()
    for name in ["エリクサー", "回復薬", "マナポーション"]:
        fetch_ingredient(name)
    print(f"合計 {time.perf_counter() - start:.1f} 秒")   # → 約 9 秒
```

## 非同期の世界 — 待ち時間に次の仕事

```python
import asyncio
import time

async def fetch_ingredient(name: str) -> str:      # ① async def = コルーチン
    print(f"  🧚 {name} を取りに行きます…")
    await asyncio.sleep(3)                          # ② await = 「待つ間、他をどうぞ」
    print(f"  ✅ {name} の材料が届きました")
    return f"{name} の材料"

async def serve_three() -> None:
    start = time.perf_counter()
    results = await asyncio.gather(                 # ③ 3 件を同時進行
        fetch_ingredient("エリクサー"),
        fetch_ingredient("回復薬"),
        fetch_ingredient("マナポーション"),
    )
    print(f"合計 {time.perf_counter() - start:.1f} 秒")   # → 約 3 秒!

asyncio.run(serve_three())                          # ④ イベントループを起動
```

妖精は 3 人いたのです。9 秒が 3 秒になりました。

### 実際に何が起きているか、印字順で追う

このコードを実行すると、`print` は実際にはこの順番で出力されます。

```
  🧚 エリクサー を取りに行きます…
  🧚 回復薬 を取りに行きます…
  🧚 マナポーション を取りに行きます…
（ここで約3秒、何も表示されない ―― でも店は止まっていない)
  ✅ エリクサー の材料が届きました
  ✅ 回復薬 の材料が届きました
  ✅ マナポーション の材料が届きました
合計 3.0 秒
```

3 件とも「取りに行きます」が **先に全部並び**、「届きました」は **後でまとめて** 出ています。
1 件ずつ順番に最後まで終わらせているわけではないことが、この出力だけでわかります。
`asyncio.gather` が呼ばれた瞬間、裏側では次のことが起きています。

1. `fetch_ingredient("エリクサー")` が動き出す → `🧚 …` を印字 → `await asyncio.sleep(3)` に到達。
   ここで **第10章の `yield` と同じ原理で一時停止**(凍結)し、制御をイベントループに返す
2. まだ手が空いているイベントループが、間髪入れずに `fetch_ingredient("回復薬")` を動かす →
   同様に `await` で凍結して制御を返す
3. `マナポーション` も同じく起動 → 凍結
4. この時点で 3 件とも「3 秒待ち」に入っている。イベントループは自分では何もせず、
   「3 秒経ったら起こして」と OS にタイマーを 3 つ仕掛けた状態で待つ
5. 3 秒後、3 つのタイマーがほぼ同時に発火 → イベントループが凍結していた 3 件を順に再開し、
   `await` の続き(`✅ …` の印字と `return`)を実行する
6. 3 件全部の結果が揃ったところで `asyncio.gather` が戻り、`serve_three` の続きが実行される

ポイントは、**「同時に実行されている」のではなく「1 つずつ、待つ直前まで高速に切り替えている」** ことです。
Python のコードそのものは常に 1 行ずつしか進みません。「同時」に見えるのは、
実際に時間がかかる部分(妖精が材料を取りに行く=OS のタイマーや通信)を Python の外に丸投げして、
その間に **他の待ち行列を回している** からです。

```mermaid
sequenceDiagram
    participant L as イベントループ(店主)
    participant A as 注文A エリクサー
    participant B as 注文B 回復薬
    participant C as 注文C マナポーション
    L->>A: 開始
    Note over A: await(妖精を待つ)🧚
    L->>B: Aの待ち時間に開始
    Note over B: await 🧚
    L->>C: Bの待ち時間に開始
    Note over C: await 🧚
    A-->>L: 3秒後 完了
    B-->>L: 完了
    C-->>L: 完了
    Note over L: 合計 約3秒(同期なら9秒)
```

### 文法の要点

| 構文 | 意味 |
|---|---|
| `async def f():` | コルーチン関数を定義。呼んでも **すぐには実行されない**(第10章のジェネレータと同じ!) |
| `await x` | x の完了を待つ。**待っている間、イベントループは他のコルーチンを進める** |
| `asyncio.run(main())` | イベントループを起動する入口。プログラムに 1 回だけ |
| `asyncio.gather(a, b, c)` | 複数を同時進行させ、全部の結果を待つ |
| `asyncio.create_task(f())` | 「あとで結果を受け取る」タスクとして即座に走らせ始める |

> 💡 実はコルーチンは **ジェネレータの子孫** です。`await` で「一時停止して制御を返す」
> 仕組みは、第10章の `yield` の凍結・再開とまったく同じ原理で動いています。

### async def と await、主語はどっち?

「`async def` で定義すると、途中で `await` を呼べるようになる」という理解で **ほぼ正解** ですが、
主従が逆になりがちなので整理します。

- **主役は `async def`**: 「この関数は、実行の途中で一時停止して、また後で再開できる特殊な体質にする」
  という宣言です。第10章の `yield` があるだけで関数がジェネレータになったのと同じで、
  `async def` は関数を **中断可能なコルーチン** に作り変えます
- **`await` はその一時停止ボタン**: 「中断可能な体質」を持つフレームの中でしか押せないボタンなので、
  Python は `async def` の外で `await` を書くこと自体を構文エラーにしています

つまり `await` は `async def` の **おまけ機能** ではなく、`async def` が用意した
「一時停止できる場所」に実際に一時停止命令を置くための唯一の手段、という関係です。

### await の右側に置けるのは async def の関数だけ?

これは分けて考える必要があります。

- **`await` を「書ける場所」** = `async def` の中だけ(これは今説明した通り)
- **`await` の「右側に置けるもの」** = `async def` の関数呼び出しだけとは限りません

正確なルールは「**`__await__` という dunder メソッドを持つオブジェクト(= awaitable)** なら何でも
`await` できる」です。これは第9章の `__iter__` があれば for で回せる(iterable)、
第12章の `__enter__`/`__exit__` があれば with で使える(context manager)と **全く同じパターン** です。
`for` / `with` / `await` は、Python では毎回「特定の dunder を持っているかどうか」で対象を判定しています。

具体的に awaitable には主に3種類あります。

| 種類 | 例 | 備考 |
|---|---|---|
| コルーチンオブジェクト | `fetch_ingredient("x")` の戻り値 | `async def` を呼ぶと自動的にこれになる。一番よく使う |
| `asyncio.Future` | `loop.create_future()` | 「まだ結果のない箱」。前節の `asyncio.sleep` の内部で実際に使われていたのがコレ |
| `asyncio.Task` | `asyncio.create_task(f())` | `Future` の子孫。実行中のコルーチンの進捗を表す |

実は前節の「裏側の実装」ですでに1つ出会っています ―― `asyncio.sleep(3)` の中身は、
`async def` で定義された関数ではあるものの、最終的に `await` している相手は `Future` オブジェクトで、
`Future.__await__` の中に生の `yield self` が書かれています。`async def` はあくまで
「`__await__` を自動生成してもらうための一番手軽な書き方」であって、`await` できるものの正体は
**「`__await__` を持っているかどうか」だけ** なのです。

普段アプリケーションコードを書く分には「`await` は `async def` の関数(かそれが返すコルーチン)に使うもの」
と思って 99% 困りません。`Future` / `Task` を意識するのは、`asyncio` ライブラリの内部や、
低レベルな独自ライブラリを書くときくらいです。

### await を1回も使わない async def に意味はあるか

ほぼありません。`async def` だけあって中に `await` が1つもない関数は:

```python
async def useless() -> int:
    total = 0
    for i in range(1000):
        total += i * i          # 時間のかかる I/O 待ちが一切ない
    return total

# 呼び出し側は結果を得るためにやっぱり await が必要
result = await useless()        # 同期関数を呼ぶのと体感速度は変わらない
```

- 呼んでもコルーチンオブジェクトが返るだけで、`await` してあげないと中身は実行されない
- 中断ポイント(`await`)が一度もないので、動き出したら **最後まで一気に実行される** ―― 普通の
  `def` と処理内容は同じなのに、呼び出し側に `await` という手間だけを増やしている
- 唯一意味があるのは、インターフェース上「この関数は `async def` でなければならない」と決まっている場合
  (抽象メソッドのオーバーライドなど)や、**条件によっては** `await` する分岐がある場合です

判断基準はシンプルです ―― **「待ち時間」が本当にある処理だけを `async def` にする**。
待ち時間が一切ない処理を `async def` にしても、非同期処理の恩恵(他の仕事に切り替える隙間)は
生まれず、ただの回りくどい `def` になるだけです。

### 具体例: データベースを await でクエリする

「実務ではどんな処理で使われるのか」で挙げた DB 問い合わせを、実際のコードで見てみます。
PostgreSQL 用の非同期ドライバ `asyncpg` を使った例です(`aiomysql` / `aiosqlite` でも考え方は同じ)。

```python
import asyncio
import asyncpg

async def fetch_potion(pool: asyncpg.Pool, name: str) -> asyncpg.Record:
    async with pool.acquire() as conn:      # ← "async with"!(後述)
        return await conn.fetchrow(         # ← クエリの実行そのものがI/O待ちなので await
            "SELECT name, price, stock FROM potions WHERE name = $1", name,
        )

async def main() -> None:
    pool = await asyncpg.create_pool(dsn="postgresql://user:pass@localhost/shop")  # 接続確立もawait
    try:
        # 3商品ぶんの問い合わせを同時に投げて、まとめて待つ(この章のはじめの例と同じ形!)
        rows = await asyncio.gather(
            fetch_potion(pool, "回復薬"),
            fetch_potion(pool, "マナポーション"),
            fetch_potion(pool, "エリクサー"),
        )
    finally:
        await pool.close()                  # 後片付けもI/O待ちなので await

asyncio.run(main())
```

ポイントは **「時間のかかる操作のほぼ全部に `await` が付く」** ことです。接続を張るのも、
クエリを送って応答を待つのも、切断するのも、すべて「妖精を待つ」と同じ構造の待ち時間だからです。

**注意点が1つ**: 世の中でよく使われる `sqlite3` / `psycopg2` / `PyMySQL` などの **定番 DB ドライバは
非同期対応していません**。これらの `cursor.execute()` は普通の同期関数なので、`async def` の中で
呼んでしまうと前述の掟②の通り `time.sleep` と同じ扱いになり、**イベントループごと全員停止**します。
非同期でDBを扱うには、`asyncpg` / `aiomysql` / `aiosqlite`、あるいは SQLAlchemy 2.0 の非同期モードのような
**async 対応のドライバ・ORM** を選ぶ必要があります。

**`async with` にも気づきましたか?** これは `with` の非同期版です。
第12章の `with` が `__enter__` / `__exit__` を呼ぶように、`async with` は
`__aenter__` / `__aexit__`(先頭に `a` が付くだけ)を **`await` しながら** 呼びます。
「コネクションプールから接続を借りる」こと自体がネットワーク越しの操作で待ち時間を伴う場合があるため、
借りる/返す処理そのものも `await` できる必要がある ―― それが `async with` です。
(同様に、非同期版の for も存在し `async for` と書きます。`__aiter__` / `__anext__` を使う版です。)
`for`→`__iter__`、`with`→`__enter__`/`__exit__`、そして `async with`→`__aenter__`/`__aexit__` ――
**すべて同じ「特定の dunder を持っているかで対象を判定する」という Python の一貫したパターン**です。

### 裏側の実装 — 「制御を返す」の正体を自作してみる

「制御を返す」は比喩ではなく、実際に `yield` が **値を1個、呼び出し元に手渡している** だけです。
第10章で見た「`next()` すると `yield` まで進んで凍結する」を、そのままイベントループの材料にできます。
`asyncio` を使わず、**素の `yield` だけ** でミニ・イベントループを自作してみましょう。

```python
import time

def fetch_ingredient_gen(name):
    print(f"  🧚 {name} を取りに行きます…")
    wake_at = time.perf_counter() + 3
    yield wake_at            # ← 「制御を返す」の正体: “3秒後に起こして” という値を渡すだけ
    print(f"  ✅ {name} の材料が届きました")

def tiny_event_loop(tasks):
    """本物のイベントループの骨格を、10行で再現したもの"""
    waiting = {task: next(task) for task in tasks}   # ① 全員を1回動かし、最初の yield 値(起床時刻)を集める
    while waiting:
        now = time.perf_counter()
        for task, wake_at in list(waiting.items()):
            if now >= wake_at:                        # ② 起床時刻を過ぎているタスクを見つけたら
                del waiting[task]
                try:
                    wake_at = task.send(None)           # ③ 凍結地点(yieldの次)から再開させる
                    waiting[task] = wake_at             #    まだ続きがあれば、次の起床時刻を登録
                except StopIteration:
                    pass                                 # 完走したタスクは管理対象から外れる

tasks = [fetch_ingredient_gen(n) for n in ["エリクサー", "回復薬", "マナポーション"]]
tiny_event_loop(tasks)
```

これで実際に3件が「9秒」ではなく「3秒」で終わります。ポイントは:

- **`task` の正体はただのジェネレータ**。`next(task)` / `task.send(None)` を呼べるだけの、ごく普通のオブジェクトです
- `yield wake_at` は「一時停止する」と同時に、**「いつ起こしてほしいか」という情報を呼び出し元(イベントループ)に渡している**
- イベントループは `waiting` という **dict(タスク → 起床時刻)を持っているだけ** の管理者です。
  自分では何も「待つ」処理をせず、ひたすら「今起こすべきタスクはどれか」を見て `.send(None)` するだけ
- `.send(None)` された側は、`yield` の次の行から実行を再開する ―― これが「凍結地点から再開」の実体です

**本物の `asyncio` も骨格はこれと全く同じ** です。違いは主に2点だけ:

1. `await asyncio.sleep(3)` は、実は内部で `yield` **していない**ように見えますが、`async/await` は
   コンパイル時にジェネレータとほぼ同じバイトコードに変換される特殊構文です。突き詰めると
   `asyncio.sleep` の中には `Future` という「まだ結果のない箱」を `yield` する1行が隠れていて、
   `await` はその `yield` を(何段 `await` が重なっていても)**一番外側のイベントループまで
   バケツリレーで届ける** 役目をしています(内部的には「`yield from` の親戚」だと考えてください)
2. 上のミニ版は `while True` で **時計を見張り続ける**(ビジーウェイト)ため CPU を無駄に消費しますが、
   本物のイベントループは OS の `select` / `epoll` という仕組みに
   「次に何かが起きるまで、この時刻(またはこのソケットにデータが届くまで)スリープしていい」と
   丸投げします。OS が起こしてくれるまで **CPU をまったく使わずに待てる** のが本物との違いです

つまり `await` で「制御がイベントループに返る」とは、`yield` された値(「いつ/何を待っているか」という情報)が
呼び出し元(イベントループ)まで届き、イベントループがそれを `waiting` のような台帳に記録して、
条件が満たされたときに **同じジェネレータの `.send()` を呼び直す** ―― という、驚くほど単純な仕組みなのです。

### ⚠️ 非同期の掟

1. `await` できるのは `async def` の中だけ
2. コルーチンの中で `time.sleep(3)` を呼ぶと **イベントループごと全員停止** します。
   非同期対応版(`asyncio.sleep`、HTTP なら `aiohttp` / `httpx`)を使うこと

   ```python
   async def bad_fetch(name: str) -> str:
       time.sleep(3)          # ❌ これは「凍結して制御を返す」通常の一時停止ではない
       return name
   # time.sleep は「待つ間、他をどうぞ」を知らない普通の関数。
   # 店主(=たった1本しかないイベントループ)自身の足を止めてしまうので、
   # 他の注文は 1 件たりとも進まなくなる ―― 3件実行すれば結局 9 秒かかる
   ```

3. 「呼び忘れ」に注意: `fetch_ingredient("x")` と書いて `await` を忘れると、
   コルーチンオブジェクトが作られるだけで実行されません(警告が出ます)

   ```python
   fetch_ingredient("回復薬")           # ❌ 妖精は送り出されない。<coroutine object ...> が
                                        #    ただ作られて、誰にも実行されず放置される
   await fetch_ingredient("回復薬")     # ✅ これで初めて実行される
   ```
   第10章の「`brew(...)` を呼んだだけでは中身が1行も実行されない」のと全く同じ話です。
   `async def` の関数を呼ぶ行為は「実行する」ではなく「予約する」だけなのです。

## async はいつ効くのか — I/O バウンド vs CPU バウンド

非同期が速くしたのは「**待ち時間**」であって「計算」ではありません。仕事は 2 種類あります。

- **I/O バウンド**: 通信・ファイル・DB の応答待ちが大半(妖精待ち)→ **async が効く**
- **CPU バウンド**: 計算そのものが重い(巨大な釜をかき混ぜ続ける)→ async では速くならない

### 見分け方 — どちらのタイプか判断する簡単な方法

コードを見て「これは待ち時間が主役?計算が主役?」と迷ったら、こう考えます。

> **その処理の間、CPU の使用率はどうなっているか?**
> - ほぼ 0%(何もせず座って待っているだけ)→ **I/O バウンド**
> - 100% 近く張り付く(ずっと計算し続けている)→ **CPU バウンド**

`fetch_ingredient` の `await asyncio.sleep(3)` は、3秒間 CPU を一切使いません(妖精を待つだけ)。
一方 `heavy_brew` の `sum(i * i for i in range(10_000_000))` は、その間ずっと CPU をフル稼働させます。
これが async が効く/効かないの分かれ目です ―― **async は「CPU が暇な時間」を他の仕事に回す技術** なので、
そもそも CPU が暇にならない CPU バウンドな処理には何もしてあげられません。

### 実シナリオ別: どの処理がどちらに当てはまるか

| シナリオ | 分類 | 理由 | 向いている技術 |
|---|---|---|---|
| Web API を100件叩いて集計 | I/O バウンド | ほとんどの時間がネットワーク応答待ち | `asyncio` |
| DB に複数クエリを投げて結果を待つ | I/O バウンド | クエリ実行自体は DB 側の仕事、Python は待つだけ | `asyncio` |
| ファイルを1000個読み込んで中身を集める | I/O バウンド | ディスク読み込み待ちが支配的 | `asyncio` (`aiofiles`) |
| チャットボットが複数ユーザーに応答 | I/O バウンド | メッセージ受信待ちがほとんど | `asyncio` |
| 画像を1000枚リサイズする | **CPU バウンド** | 画素の計算がずっと CPU を使い続ける | `multiprocessing` |
| 動画のエンコード | **CPU バウンド** | エンコード計算そのものが重い | `multiprocessing` |
| パスワードハッシュ化(bcrypt など) | **CPU バウンド** | 意図的に計算コストを高くしてある | `multiprocessing`(または専用スレッド) |
| 巨大な CSV を集計・変換する | **CPU バウンド寄り** | 読み込みは I/O だが、集計計算が支配的になりがち | データ量次第。まず計測する |
| 機械学習モデルの推論(ローカル実行) | **CPU/GPU バウンド** | 行列計算そのものが重い | `multiprocessing`(GPU 利用ならまた別の話) |
| 外部の推論 API(例: 生成 AI API)を呼ぶ | I/O バウンド | 計算は相手のサーバーがやっている。手元は応答待ちだけ | `asyncio` |

同じ「画像を扱う」でも、**「自分の CPU で計算するか」「相手に計算してもらって応答を待つだけか」**
で分類が変わる点に注目してください。判断基準は常に「自分の CPU が忙しいか、暇なのに待たされているか」です。

### 現実によくある「混在パターン」への対処

Web サーバーなど実際のアプリでは、I/O バウンドな処理の中に **時々 CPU バウンドな処理が混ざります**。
例えば「注文を受ける(I/O)→ 画像サムネイルを生成する(CPU)→ 結果を返す(I/O)」のような場合です。

ここで CPU バウンドな部分を `async def` の中で普通に計算してしまうと、掟②の通り
**その計算が終わるまでイベントループごと止まり**、他のお客さんの処理も一切進まなくなります。
対処法は、CPU バウンドな部分だけを別プロセス(または別スレッド)に **逃がす** ことです。

```python
import asyncio
from concurrent.futures import ProcessPoolExecutor

def make_thumbnail(image_path: str) -> str:
    # ここは普通の同期関数でOK。重い画像処理をする
    ...
    return f"{image_path}.thumb.jpg"

async def handle_order(pool: ProcessPoolExecutor, image_path: str) -> str:
    loop = asyncio.get_running_loop()
    # 重い処理だけを別プロセスに投げ、その間イベントループは他の仕事を続けられる
    thumbnail = await loop.run_in_executor(pool, make_thumbnail, image_path)
    return thumbnail
```

`loop.run_in_executor(pool, 関数, 引数...)` は「この同期関数をプールに任せて、終わったら教えて」を
`await` 可能な形にしてくれる橋渡し役です。**「I/O は asyncio で、CPU 計算だけ pool に逃がす」**
という組み合わせは、実務の非同期 Web サーバー(FastAPI など)で非常によく使われるパターンです。

### GIL — Python 界の有名な門番

「じゃあ CPU バウンドはスレッドで並列に?」— ここで **GIL(グローバルインタプリタロック)**
の話をせねばなりません。CPython では **同時に Python バイトコードを実行できるスレッドは
常に 1 つだけ** です。つまりスレッドを 8 本立てても、CPU 計算は速くなりません。

CPU バウンドを本当に並列にするには **プロセス** を分けます(`multiprocessing` /
`concurrent.futures.ProcessPoolExecutor`)。プロセスごとに独立した Python が立つので
GIL の制約を受けません。

```mermaid
flowchart TD
    Q1{"仕事の種類は?"} -->|"待ち時間が主役<br/>(通信・ファイル・DB)"| Q2{"接続数は?"}
    Q1 -->|"計算が主役<br/>(数値計算・画像処理)"| P["multiprocessing 🏭<br/>プロセス並列(GILを回避)"]
    Q2 -->|"多い(数百〜)"| A["asyncio ⚡<br/>1スレッドで大量の待ちをさばく"]
    Q2 -->|"少ない/ライブラリが同期式"| T["threading 🧵<br/>お手軽・GILは待ち中に解放される"]
```

| 方式 | 並行の単位 | 得意分野 | ひとこと |
|---|---|---|---|
| `asyncio` | コルーチン(超軽量) | 大量の I/O 待ち | 協調的。`await` で自主的に譲り合う |
| `threading` | OS スレッド | 少数の I/O 待ち、同期ライブラリ | I/O 待ち中は GIL が解放されるので有効 |
| `multiprocessing` | OS プロセス | CPU 計算 | 起動もデータ受け渡しも重いが真の並列 |

```python
# CPU バウンドをプロセス並列にする例(釜 4 つで同時に煮込む)
from concurrent.futures import ProcessPoolExecutor

def heavy_brew(potion_id: int) -> str:
    total = sum(i * i for i in range(10_000_000))   # 重い計算
    return f"potion-{potion_id} 完成"

if __name__ == "__main__":
    with ProcessPoolExecutor(max_workers=4) as pool:      # 第12章の with!
        for result in pool.map(heavy_brew, range(8)):
            print(result)
```

## 🧪 完成コード: `shop/async_shop.py` — 行列を同時にさばく

```python
"""Pythonic Potions — 14 日目: 非同期店舗"""

import asyncio
import random

async def serve(customer: str, item: str) -> int:
    print(f"🙋 {customer} さん: {item} をください")
    brewing_time = random.uniform(0.5, 3.0)
    await asyncio.sleep(brewing_time)                # 醸造待ち(他の接客に譲る)
    print(f"🧪 {customer} さんへ {item} をお渡し({brewing_time:.1f}秒)")
    return 50

async def open_shop() -> None:
    queue: asyncio.Queue[tuple[str, str]] = asyncio.Queue()

    # 3 人の店員(ワーカー)が同じ行列(キュー)からお客さんを取る
    async def clerk(clerk_id: int) -> None:
        while True:
            customer, item = await queue.get()
            await serve(customer, item)
            queue.task_done()

    clerks = [asyncio.create_task(clerk(i)) for i in range(3)]

    for i, (customer, item) in enumerate([
        ("アリス", "回復薬"), ("ボブ", "エリクサー"), ("キャロル", "マナポーション"),
        ("デイブ", "回復薬"), ("イブ", "解毒薬"),
    ]):
        await queue.put((customer, item))

    await queue.join()               # 行列がはけるまで待つ
    for c in clerks:
        c.cancel()                   # 店員さん、お疲れさまでした

if __name__ == "__main__":
    asyncio.run(open_shop())
```

実行すると、5 人のお客さんが 3 人の店員に **同時並行で** さばかれていく様子が見えます。
`asyncio.Queue` + ワーカーは、実世界の非同期アプリ(クローラー、チャットボット、
API サーバー)の最頻出パターンです。

## 📝 今日の開店準備(演習)

1. `serve` に「エリクサーは醸造に必ず 5 秒かかる」を追加し、店員数を 1 / 3 / 5 人に変えて合計時間を比べてください。
2. `asyncio.wait_for(serve(...), timeout=2.0)` で「2 秒以上待たせたら謝ってキャンセル」を実装してください(`TimeoutError` を捕まえる — 第6章!)。
3. `heavy_brew` を `ThreadPoolExecutor` と `ProcessPoolExecutor` の両方で走らせ、実行時間を計測して GIL の効果を体感してください。

---

お店は行列もさばける大繁盛店に。最終開発章では、Python が **クラスを作る仕組みそのもの** に
手を入れて、「新作ポーションを書くだけで自動的に店に並ぶ」プラグイン機構を作ります
→ [第15章 プラグインで無限拡張](15_metaprogramming.md)
