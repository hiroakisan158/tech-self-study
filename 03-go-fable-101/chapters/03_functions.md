# 第3章 仕分け係を雇う — 関数・多値返却・defer

## 🚇 今日のお話

荷物が増えて所長ひとりでは回らなくなりました。運賃計算や仕分けの手順を
**関数** として文書化し、仕分け係のゴーファーに任せます。

Go の関数には Python にない大きな特徴が 2 つあります。
**値を複数返せる** ことと、**defer** という後片付け予約の仕組みです。
この 2 つが、後の章のエラー処理とリソース管理の土台になります。

## 関数の基本形

```go
// func 名前(引数 型) 戻り値の型 { }
func calcFare(weight float64) int {
	return 500 + int(weight*20)
}

func main() {
	fare := calcFare(18.5)
	fmt.Println(fare) // 870
}
```

- 引数と戻り値の型は **必須** です(Python の型ヒントと違い省略不可)
- 同じ型の引数が並ぶときは `func f(x, y float64)` とまとめられます

> 🐍 **Python との違い①: デフォルト引数もキーワード引数もない**
> `def f(x, retries=3, *, verbose=False)` のような柔軟な引数は Go にありません。
> 引数は「全部・順番通り・型通り」に渡すだけです。
>
> 🔍 **なぜないの?** デフォルト引数は便利ですが、引数が増えるたびに
> 「どの組み合わせが有効か」が爆発し、API が肥大化しがちです。Go は
> 「引数が多すぎるなら struct を渡す」「バリエーションが要るなら別名の関数を作る」
> ことを選びました。標準ライブラリの `strings.Replace` と `strings.ReplaceAll` が
> 別関数なのはこの思想です。オプションが本当に多い場合は Functional Options
> パターン(第16章)が使われます。

## 多値返却 — Go 流エラー処理の入り口

Go の関数は値を複数返せます。そして **「結果とエラーのペア」を返すのが Go の基本作法** です。

```go
func splitParcels(total, boxSize int) (int, int) {
	return total / boxSize, total % boxSize // 箱数と余り
}

boxes, rest := splitParcels(32, 5) // 6, 2
```

本命はこちらの形です:

```go
import "errors"

func findFare(dest string) (int, error) {
	fares := map[string]int{"north": 500, "south": 700}
	fare, ok := fares[dest]
	if !ok {
		return 0, errors.New("未開通の行き先: " + dest)
	}
	return fare, nil
}

func main() {
	fare, err := findFare("east")
	if err != nil {
		fmt.Println("受付できません:", err)
		return
	}
	fmt.Println("運賃:", fare)
}
```

この `if err != nil` は Go プログラムで最も頻出するイディオムです。

> 🐍 **Python との違い②: 例外を投げない、エラーを返す**
> Python なら `raise ValueError(...)` と例外を投げ、呼び出し側の好きな場所で
> `try/except` しました。Go では **エラーはただの戻り値** で、呼んだその場で
> 毎回処理(または上に返送)します。
>
> 🔍 **なぜ例外がないの?** 例外は「関数の見た目に現れない隠れた出口」を作ります。
> どの行で大域脱出が起きるか読み手には分からず、Go の設計者はこれを
> 「制御フローが見えなくなる」として退けました。エラーを戻り値にすると、
> **失敗しうる関数はシグネチャを見れば分かり、無視するにはコードに痕跡が残ります**。
> 冗長さと引き換えに「エラー処理が本流のコードと同じ場所に見える」ことを
> 選んだのです。詳しい設計技法は第10章で扱います。

## 名前付き戻り値と naked return

戻り値に名前を付けられます。ドキュメントとして有用ですが、`return` を裸で書く
「naked return」は短い関数以外では避けるのが慣習です(何が返るのか追いにくくなるため)。

```go
func splitParcels(total, boxSize int) (boxes, rest int) {
	boxes = total / boxSize
	rest = total % boxSize
	return // boxes, rest が返る(短い関数ならOK、長い関数では嫌われる)
}
```

## defer — 後片付けの予約

`defer` を付けた呼び出しは、**関数を抜けるとき**(return 後・panic 時も)に実行されます。

```go
func deliver() {
	fmt.Println("トンネルのゲートを開ける")
	defer fmt.Println("トンネルのゲートを閉める") // ← 予約しておく

	fmt.Println("荷物を配達中...")
	// 途中で return しても、ゲートは必ず閉まる
}
// 出力: 開ける → 配達中 → 閉める
```

典型的な使い方はリソースの解放です:

```go
f, err := os.Open("manifest.txt")
if err != nil {
	return err
}
defer f.Close() // 開いた直後に閉じる予約を書く
// ... f を使う処理が何行続いても、closeし忘れない
```

> 🐍 **Python との違い③: `with` の代わりが `defer`**
> Python の `with open(...) as f:` はブロックを抜けると自動で閉じました。
> Go の `defer` は **関数を** 抜けるときに動きます。ブロック単位ではないことに注意
> (ループ内で defer を積むと関数終了までたまり続けます)。
> `with` は専用プロトコル(`__enter__`/`__exit__`)が必要でしたが、defer は
> どんな関数呼び出しでも予約できる、より素朴で汎用的な道具です。

### ⚠️ defer の 2 大落とし穴

**① 引数は defer した瞬間に評価される**

```go
count := 1
defer fmt.Println("最終個数:", count) // ← ここで count=1 が確定
count = 99
// 出力は「最終個数: 1」!
```

後で評価してほしいときはクロージャで包みます: `defer func() { fmt.Println(count) }()`

**② 複数の defer は LIFO(後入れ先出し)**

```go
defer fmt.Println("1 番目の予約")
defer fmt.Println("2 番目の予約")
// 出力: 2 番目 → 1 番目
```

「開けた順と逆に閉める」ためのスタック構造です。ゲート A → ゲート B と開けたら、
B → A の順で閉めるのが自然、という理屈です。

## 関数は値 — クロージャ

Python 同様、Go の関数も第一級の値です。変数に入れ、引数で渡し、
**周囲の変数を捕まえた関数(クロージャ)** を作れます。

```go
// 連番の伝票番号を発行するカウンタを作る
func newCounter() func() int {
	n := 0
	return func() int { // n を捕まえたクロージャ
		n++
		return n
	}
}

next := newCounter()
fmt.Println(next(), next(), next()) // 1 2 3
```

> 🐍 Python でクロージャの変数を書き換えるには `nonlocal n` が必要でしたが、
> Go では宣言(`:=`)と代入(`=`)が別の構文なので、曖昧さがなくそのまま書けます。
> なお Go には `lambda` に相当する省略記法はなく、無名関数も `func(x int) int { ... }` と
> フルで書きます。デコレータ構文もありませんが、「関数を受け取って関数を返す」こと自体は
> 普通にできます(演習 3)。

## 🚇 完成コード: `express/day3/main.go`

```go
// Gopher Express — 仕分け係の業務マニュアル
package main

import (
	"errors"
	"fmt"
)

var fares = map[string]int{"north": 500, "south": 700, "west": 650}

// 行き先から運賃を調べる。未開通ならエラー。
func findFare(dest string, weight float64) (int, error) {
	base, ok := fares[dest]
	if !ok {
		return 0, errors.New("未開通の行き先: " + dest)
	}
	return base + int(weight*20), nil
}

// 伝票番号の発行係(クロージャ)
func newTicketIssuer() func() string {
	n := 0
	return func() string {
		n++
		return fmt.Sprintf("GX-%04d", n)
	}
}

func main() {
	defer fmt.Println("🔒 営業所を施錠しました")

	issue := newTicketIssuer()
	orders := []struct {
		dest   string
		weight float64
	}{
		{"north", 3.2}, {"east", 1.0}, {"south", 42.0},
	}

	for _, o := range orders {
		fare, err := findFare(o.dest, o.weight)
		if err != nil {
			fmt.Println("⚠️ 受付不可:", err)
			continue
		}
		fmt.Printf("✅ %s %s 行き %d ゴールド\n", issue(), o.dest, fare)
	}
}
```

## 📝 今日の配達訓練(演習)

1. 可変長引数 `func totalWeight(weights ...float64) float64` を書き、
   `totalWeight(3.2, 42.0, 0.4)` の合計を返してください(Python の `*args` 相当です)。
2. `defer` を 3 つ積んで実行順を確認してください。さらに「引数は予約時に評価」の
   落とし穴を自分のコードで再現し、クロージャ版で直してください。
3. デコレータもどき: `func logged(f func(float64) int) func(float64) int` を書き、
   呼び出し前後にログを印字する関数でラップしてください。Python のデコレータとの
   書き味の違い(`@` 構文がない、型を全部書く)を体感しましょう。

---

仕分け係は雇えましたが、荷物を 1 個ずつ変数で持つのは限界です。荷物をまとめて運ぶ
**コンテナ** が必要です。Go のスライスは Python のリストに似て非なるもの——
この教材で最初の「本気の落とし穴」章です。 → [第4章 荷物コンテナ](04_slices.md)
