# 第13章 管制室 — select・context・sync・race検出

## 🚇 今日のお話

配達員は増えましたが、管制室がありません。今日は実務の並行処理に必要な
管制設備を一式導入します:

- **select** — 複数のトンネルを同時に見張るモニター
- **context** — 「全員、作業中止して帰社せよ」の一斉放送
- **sync.Mutex** — どうしても共有したい帳簿の鍵
- **race detector** — 事故を検出するドライブレコーダー

## select — 複数トンネルの同時見張り

`select` は「複数の channel 操作のうち、**先に準備できたもの** を実行する」文です。
switch に似ていますが、条件ではなく **通信** で分岐します。

```go
select {
case o := <-northTunnel:
	fmt.Println("北から荷物:", o)
case o := <-southTunnel:
	fmt.Println("南から荷物:", o)
case <-time.After(3 * time.Second):
	fmt.Println("3 秒間何も来ない。見回りに行く")
}
```

- 複数同時に準備できていたら **ランダムに 1 つ** 選ばれます(公平性のため。
  map の反復順ランダム化と同じ「依存させない」思想です)
- `default` 節を書くと「どれも準備できてないなら待たずに抜ける」になります
- `for { select { ... } }` で回し続けるのが管制室の基本形です

`time.After` との組み合わせは **タイムアウト** の基本イディオムです。
Python の `asyncio.wait_for` に相当します。

## context — 一斉帰社命令

前章の宿題「goroutine リーク」の解決策です。Go の実務コードで最頻出の型、
`context.Context` は **「キャンセルと締切を運ぶ伝達網」** です。

```go
func patrol(ctx context.Context, name string) {
	for {
		select {
		case <-ctx.Done(): // 帰社命令が来たら
			fmt.Println(name, "帰社:", ctx.Err())
			return // ← ここで確実に終了するので、リークしない
		case <-time.After(200 * time.Millisecond):
			fmt.Println(name, "巡回中")
		}
	}
}

func main() {
	ctx, cancel := context.WithTimeout(context.Background(), time.Second)
	defer cancel() // 使い終わったら必ず cancel(リソース解放)

	go patrol(ctx, "ゴン")
	go patrol(ctx, "ファー")

	<-ctx.Done() // 1 秒後に全員に Done が届く
	time.Sleep(100 * time.Millisecond)
}
```

- `context.Background()` が根っこ。`WithTimeout` / `WithCancel` / `WithDeadline` で
  枝分かれさせ、**親を cancel すると子孫全員に伝播** します
- 慣習: **ctx は関数の第 1 引数** に取り、struct に保存しない

> 🔍 **なぜ「第 1 引数に ctx」なんて原始的な方法なの?**
> Python の asyncio はタスクに `task.cancel()` を外から打ち込めました。Go の
> goroutine には **ハンドルも ID もなく、外から殺す手段が意図的にありません**。
> 強制終了は「ロックを持ったまま殺される」等の地獄を生むからです(Java の
> Thread.stop が廃止された歴史)。Go の答えは「**殺さない。本人に伝えて、
> 本人に安全な場所で辞めてもらう**」——その伝達手段が context です。
> 全関数に ctx を引き回すのは確かに不格好ですが、「この関数はキャンセル可能で、
> 締切を尊重する」ことがシグネチャから読める、という Go らしい明示性でもあります。

## sync.Mutex — 共有帳簿の鍵

channel が原則とはいえ、「全員が 1 冊の売上帳に書き込む」ような単純な共有には
ロックの方が素直です。

まず事故を見ます:

```go
var revenue int
var wg sync.WaitGroup
for i := 0; i < 1000; i++ {
	wg.Add(1)
	go func() {
		defer wg.Done()
		revenue += 100 // 💥 データ競合!
	}()
}
wg.Wait()
fmt.Println(revenue) // 100000 にならないことがある(97800 とか)
```

`revenue += 100` は「読む→足す→書く」の 3 手であり、2 匹が同時にやると
片方の加算が消えます。**Mutex(相互排他ロック)** で「帳簿は 1 匹ずつ」にします。

```go
type Cashbox struct {
	mu      sync.Mutex
	revenue int
}

func (c *Cashbox) Deposit(amount int) {
	c.mu.Lock()
	defer c.mu.Unlock() // Lock したら defer Unlock、が鉄板イディオム
	c.revenue += amount
}
```

> 🐍 **Python との違い①: GIL という保険がない**
> CPython では GIL のおかげで「たまたま壊れなかった」共有変更が結構ありました
> (それでも `+=` は競合しえますが、遭遇率は低い)。Go は本物の並列なので、
> **保護なしの共有書き込みは確実に事故ります**。しかも競合の結果は
> 「たまに数字がズレる」という最悪の形で現れます。
> 単純なカウンタなら `sync/atomic` パッケージ、一度きりの初期化なら `sync.Once`
> という専用道具もあります。

## -race — ドライブレコーダーを常備する

データ競合は目視でほぼ見つけられません。Go には **race detector** が標準装備です。

```bash
go run -race main.go
go test -race ./...
```

競合が起きると、**どの goroutine がどの行で読み書きしたか** まで報告してくれます。
実行が数倍遅くなるので本番では外しますが、**テストでは常に -race を付ける** のが
Go の常識です。CI に必ず入れましょう。

> 🔍 race detector は Google の ThreadSanitizer 技術の統合で、「並行処理を
> 言語の売りにするなら、事故検出器も標準で持つべき」という設計判断です。
> 「-race で一度も検出されていない並行コード」以外を信用しない癖をつけてください。

## パターン集 — 管制室の定石

### ① 完了通知だけの channel

```go
done := make(chan struct{}) // 空 struct = 「中身に意味はない、合図だけ」
go func() {
	// ... 作業 ...
	close(done) // close は全受信者に一斉に届く放送になる
}()
<-done
```

### ② ファンアウト / ファンイン

前章のワーカープールがファンアウト(1 つのトンネルから N 匹が取る)。
逆に N 匹の結果を 1 つのトンネルに集めるのがファンイン。両方組むと
「注文 → 並行処理 → 結果集約」のパイプラインになります(完成コード参照)。

### ③ errgroup — WaitGroup + エラー + context の三点セット

実務では準標準ライブラリの `golang.org/x/sync/errgroup` が定番です:
「どれか 1 つ失敗したら全員キャンセルして、最初のエラーを返す」が数行で書けます。

```go
g, ctx := errgroup.WithContext(ctx)
for _, dest := range dests {
	g.Go(func() error { return deliverTo(ctx, dest) })
}
if err := g.Wait(); err != nil { ... } // 最初のエラー
```

## 🚇 完成コード: 管制室つき配達パイプライン

```go
// Gopher Express — 管制室の稼働
package main

import (
	"context"
	"fmt"
	"math/rand"
	"sync"
	"time"
)

type Result struct {
	OrderID int
	Courier string
}

func courier(ctx context.Context, name string, orders <-chan int, results chan<- Result) {
	for {
		select {
		case <-ctx.Done():
			fmt.Printf("🐹 %s: 帰社命令を受信\n", name)
			return
		case id, ok := <-orders:
			if !ok {
				return // 受付終了
			}
			time.Sleep(time.Duration(rand.Intn(300)) * time.Millisecond)
			results <- Result{OrderID: id, Courier: name}
		}
	}
}

func main() {
	ctx, cancel := context.WithTimeout(context.Background(), 800*time.Millisecond)
	defer cancel()

	orders := make(chan int)
	results := make(chan Result)

	// ファンアウト: 3 匹が同じ注文トンネルを共有
	var wg sync.WaitGroup
	for _, name := range []string{"ゴン", "ファー", "モグ"} {
		wg.Add(1)
		go func() {
			defer wg.Done()
			courier(ctx, name, orders, results)
		}()
	}

	// 全員が帰ったら結果トンネルを閉める(ファンインの close 定石)
	go func() {
		wg.Wait()
		close(results)
	}()

	// 注文を流し込む係
	go func() {
		defer close(orders)
		for id := 1; id <= 20; id++ {
			select {
			case orders <- id:
			case <-ctx.Done():
				fmt.Println("📢 受付を打ち切り")
				return
			}
		}
	}()

	// 管制室: 結果を集約(results が閉まるまで)
	delivered := 0
	for r := range results {
		delivered++
		fmt.Printf("✅ 注文%d を %s が配達\n", r.OrderID, r.Courier)
	}
	fmt.Printf("🏢 制限時間内の配達: %d 件\n", delivered)
}
```

タイムアウトを 800ms → 5s に変えると全 20 件配達できることを確認してください。
このコードには前章までの全要素(goroutine、channel、close の掟、WaitGroup、
select、context)が詰まっています。写経の価値があります。

## 📝 今日の配達訓練(演習)

1. データ競合のコードを `go run -race` で実行し、レポートを読んでください。
   その後 Mutex 版と `sync/atomic.AddInt64` 版の両方で直してください。
2. 完成コードから `go func() { wg.Wait(); close(results) }()` を消すとどうなるか、
   予想してから実行してください(main の range が永遠に待つ = デッドロック)。
3. `select` に `default` を付けた「ノンブロッキング受信」を書き、
   「荷物があれば取る、なければ他の仕事をする」ループを作ってください。

---

社内の配達網は完成しました。次はいよいよ **外部のお客さん** とつながります。
Web から集荷依頼を受け付ける HTTP サーバーを、フレームワークなしの
標準ライブラリだけで建てます。 → [第14章 電子受付を開く](14_http_json.md)
