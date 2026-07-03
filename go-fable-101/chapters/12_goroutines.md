# 第12章 配達員とトンネル — goroutine と channel

## 🚇 今日のお話

Gopher Express の配達員はこれまで 1 匹。荷物が 100 個あれば 100 回往復していました。
今日、ついに **配達員を好きなだけ雇い(goroutine)**、配達員同士が荷物を
**トンネル(channel)** で受け渡す体制に移行します。

この章のためにこの会社はありました。ホリネズミがトンネルを掘って荷物を流す——
それが Go の並行処理のイメージそのものです。

## goroutine — `go` と言うだけで配達員が増える

```go
func deliver(id int) {
	fmt.Printf("配達員 %d: 出発\n", id)
	time.Sleep(100 * time.Millisecond) // 配達に掛かる時間
	fmt.Printf("配達員 %d: 完了\n", id)
}

func main() {
	go deliver(1) // go を付けるだけで並行実行!
	go deliver(2)
	go deliver(3)

	time.Sleep(200 * time.Millisecond) // ⚠️ 雑な待ち方(すぐ直します)
}
```

`go f()` と書くだけで、f は **goroutine** という軽量な実行の流れとして
並行に走り出します。スレッドを作る儀式も、コールバックも、`async def` もありません。

> ⚠️ **main が帰ると全員解雇**: `main` 関数が終了すると、走っている goroutine は
> 完了を待たれることなく全部消えます。上の `time.Sleep` はそれを防ぐ雑な手で、
> 正しい待ち方(WaitGroup)は後述します。

> 🐍 **Python との違い①: asyncio と goroutine は別物**
> Python 教材第14章の asyncio と比べると、似て非なるものだと分かります。
>
> | | Python asyncio | Go goroutine |
> |---|---|---|
> | 関数の色 | `async def` は別種の関数。`await` 必須 | 普通の関数を `go` で呼ぶだけ |
> | 切り替わる場所 | `await` と書いた場所だけ(協調的) | ランタイムが任意の地点で切替(先取的) |
> | CPU 並列 | ❌ シングルスレッド(+GIL) | ✅ 全 CPU コアで真の並列 |
> | うっかりブロック | ループ全体が凍る | 他の goroutine は走り続ける |
>
> asyncio の「async 関数は async からしか呼べない」という **関数の色問題** が
> Go には存在しません。ライブラリの関数が中で goroutine を使っていても、
> 呼ぶ側は普通の関数として呼ぶだけです。
>
> 🔍 **なぜそんなことが可能なの?** goroutine は OS スレッドではなく、
> Go ランタイムが管理する軽量な実行単位です(初期スタック約 2KB、必要に応じて伸縮)。
> ランタイムは少数の OS スレッドの上に多数の goroutine を載せ替え続け(M:N
> スケジューリング)、I/O 待ちに入った goroutine を外して別のを載せます。
> つまり **asyncio のイベントループ相当が言語ランタイムに内蔵されていて、
> await の挿入をコンパイラとランタイムが勝手にやってくれる** ようなものです。
> goroutine は数十万個作っても平気です。

## channel — 荷物を流すトンネル

配達員を増やすと、次の問題は **受け渡し** です。Go の答えが **channel**:
goroutine の間で値を安全に受け渡す、型付きのトンネルです。

```go
tunnel := make(chan string) // string が流れるトンネルを掘る

go func() {
	tunnel <- "回復薬" // トンネルに荷物を入れる(送信)
}()

item := <-tunnel // トンネルから荷物を取り出す(受信)
fmt.Println(item)
```

- `ch <- 値` — 送信 / `<-ch` — 受信。矢印の向き = 荷物の流れる向き
- **バッファなし channel は「手渡し」**: 送信側は受け取り手が現れるまでその場で待ち、
  受信側は荷物が来るまで待ちます。この「待ち合わせ」が同期そのものです

```go
belt := make(chan string, 3) // バッファ付き: 3 個まで置けるベルトコンベア
```

バッファ付きは「棚」です。棚に空きがあれば送信側は待たずに置いて次へ行けます。
満杯なら待ちます。

> 🔍 **なぜ共有メモリではなく channel なの?**
> 並行処理の古典的な方法は「共有変数 + ロック」ですが、ロックの取り忘れ・順序ミスは
> コンパイラで検出できず、再現困難なバグになります。Go は Tony Hoare の
> **CSP(Communicating Sequential Processes、1978)** という理論を土台に、
> **「メモリを共有して通信するな。通信によってメモリを共有せよ」**
> (Go 公式の標語)という設計を選びました。
> 荷物(データ)をトンネルに **流してしまえば、所有権ごと相手に渡る** ので、
> 同じデータを 2 匹が同時に触る状況そのものが起きにくくなるのです。
> ロック(sync.Mutex)も存在しますが(次章)、まず channel で設計を考えるのが Go 流です。

## close と range — 「今日の荷物は以上!」

送信側はトンネルを `close` して終業を伝えられます。受信側は `range` で
「閉まるまで受け取り続ける」ループが書けます。

```go
func produce(belt chan<- string) { // chan<- は「送信専用」の型(向きの明示)
	for i := 1; i <= 5; i++ {
		belt <- fmt.Sprintf("荷物%d", i)
	}
	close(belt) // 「以上!」
}

func main() {
	belt := make(chan string, 2)
	go produce(belt)

	for item := range belt { // close されるまで受け取り続ける
		fmt.Println("受領:", item)
	}
}
```

close の掟(破ると panic します):

1. **close するのは送信側**(受信側が閉めてはいけない)
2. close 済み channel への送信は panic
3. 二重 close も panic
4. close は必須ではない(range で待つ人がいるときだけ必要。放置しても GC が回収します)

closed な channel から受信すると「ゼロ値, false」が返ります:
`item, ok := <-belt` —— map と同じカンマ ok です。

## sync.WaitGroup — 全員の帰りを待つ

「N 匹の配達員が全員帰るまで待つ」の正攻法です。

```go
func main() {
	var wg sync.WaitGroup

	for i := 1; i <= 3; i++ {
		wg.Add(1) // 出発前にカウント +1(go の外で!)
		go func() {
			defer wg.Done() // 帰宅時に -1(defer で確実に)
			deliver(i)
		}()
	}

	wg.Wait() // カウントが 0 になるまで待つ
	fmt.Println("全員帰社。閉店!")
}
```

- `Add` は **goroutine を起動する前に** 呼びます(起動後だと Wait が先に通過しうる)
- Go 1.22 以降はループ変数 `i` をそのままクロージャで使えます(第2章の歴史参照)。
  古いコードの `i := i` はその名残です

## 🚇 完成コード: 並行配達システム第 1 号

「注文を作る係 1 匹 → 配達員 3 匹が同じトンネルから取っていく」——
**ワーカープール** と呼ばれる、Go 並行処理のいちばん基本的な形です。

```go
// Gopher Express — 並行配達の開始
package main

import (
	"fmt"
	"sync"
	"time"
)

type Order struct {
	ID   int
	Dest string
}

// 配達員: トンネルから注文を取り続け、閉まったら帰社
func courier(name string, tunnel <-chan Order, wg *sync.WaitGroup) {
	defer wg.Done()
	for o := range tunnel { // トンネルが close されるまで働く
		fmt.Printf("🐹 %s: 注文%d(%s 行き)を配達中\n", name, o.ID, o.Dest)
		time.Sleep(100 * time.Millisecond)
		fmt.Printf("🐹 %s: 注文%d 完了\n", name, o.ID)
	}
	fmt.Printf("🐹 %s: 帰社\n", name)
}

func main() {
	tunnel := make(chan Order, 5)
	var wg sync.WaitGroup

	// 配達員 3 匹を雇う(同じトンネルを共有)
	for _, name := range []string{"ゴン", "ファー", "モグ"} {
		wg.Add(1)
		go courier(name, tunnel, &wg)
	}

	// 受付係: 注文をトンネルに流す
	dests := []string{"north", "south", "west", "north", "south", "east", "west"}
	for i, d := range dests {
		tunnel <- Order{ID: i + 1, Dest: d}
	}
	close(tunnel) // 「本日の受付終了!」

	wg.Wait()
	fmt.Println("🏢 全配達完了。閉店!")
}
```

実行するたびに **配達順が変わる** ことを確認してください。3 匹が本当に並行に
働いている証拠です(そして「順序に依存してはいけない」warning でもあります)。

## ⚠️ この章の落とし穴ダイジェスト

```go
// ① デッドロック: 受け手のいない無バッファ channel に main が送信
ch := make(chan int)
ch <- 1 // 💥 fatal error: all goroutines are asleep - deadlock!
// → Go ランタイムは「全員寝てる」を検出してクラッシュさせてくれる(親切!)

// ② goroutine リーク: 誰も受信しない channel に送信したまま永眠する goroutine
// デッドロック検出は「全員」寝てるときだけ。1 匹だけ永眠だと検出されず漏れ続ける

// ③ WaitGroup の Add を goroutine の中でやる → Wait がすり抜ける
```

②の対策(context によるキャンセル)は次章で扱います。

## 📝 今日の配達訓練(演習)

1. 完成コードの配達員を 1 匹に減らす/10 匹に増やすと全体時間がどう変わるか、
   `time.Since` で計測してください。
2. `close(tunnel)` を消すと何が起きるか予想してから実行してください
   (配達員が range で待ち続け、wg.Wait でデッドロック検出、が正解です)。
3. 配達員が結果を返す版: `results chan string` を追加し、配達員が完了報告を流し、
   main 側で 7 件の報告を受信してください。「送信側が複数いる channel は誰が
   いつ close するか」問題に直面するはずです(ヒント: 報告件数が分かっているなら
   close せず件数分受信すればよい)。

---

配達員は増えましたが、管制が素朴すぎます。「複数のトンネルを同時に見張る」
「時間切れで打ち切る」「全配達員に一斉帰社命令を出す」——実務の並行処理に必要な
道具一式を揃えます。 → [第13章 管制室](13_concurrency_patterns.md)
