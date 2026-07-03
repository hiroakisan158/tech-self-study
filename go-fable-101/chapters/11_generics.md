# 第11章 万能コンテナ — ジェネリクス

## 🚇 今日のお話

営業所には「int 専用の合計機」「float64 専用の合計機」「重い順に並べる Parcel 専用ソーター」
……と、**中身の型だけが違う同じ設備** が増殖しています。

```go
func sumInts(xs []int) int             { ... }
func sumFloats(xs []float64) float64   { ... } // 中身は完全に同じコード
```

これを 1 台で済ませるのが **ジェネリクス(型パラメータ)** です。
Go 1.18(2022年)でようやく入った、Go 史上最大の言語拡張です。

## 型パラメータの基本形

```go
// [T any] が型パラメータ。「T は呼び出し時に決まる型」の意味
func first[T any](xs []T) T {
	return xs[0]
}

first([]int{1, 2, 3})           // T = int と推論される
first([]string{"回復薬", "紙"}) // T = string
first[float64]([]float64{1.5})  // 明示指定もできる(通常は推論に任せる)
```

> 🐍 **Python との違い①: 「動けばいい」と「証明してから動く」**
> Python の関数はもともと全部ジェネリクスのようなものでした——型を見ないので、
> どんなリストでも `def first(xs): return xs[0]` で動きます(型ヒントの
> `TypeVar` / `def first[T](xs: list[T]) -> T` は注釈に過ぎません)。
> Go のジェネリクスは注釈ではなく実体です。**「T に対してその操作が本当に
> できるか」をコンパイラが証明できた場合だけ** コンパイルが通ります。
> だから次の「制約」が必要になります。

## 制約 — T に何ができるかの契約

`any` の T には、比較も加算もできません。「T はこれができる型に限る」と
**制約(constraint)** で宣言します。制約の正体はインターフェースです。

```go
// ① 組み込みの comparable: == と != ができる型
func contains[T comparable](xs []T, target T) bool {
	for _, x := range xs {
		if x == target {
			return true
		}
	}
	return false
}

// ② 型の集合を | で列挙する(ジェネリクスと同時に入った新しい書き方)
type Number interface {
	~int | ~float64 // ~ は「これを基底とする定義型も含む」(Gold 型なども通す)
}

func sum[T Number](xs []T) T {
	var total T // T のゼロ値から開始
	for _, x := range xs {
		total += x // Number 制約のおかげで + が使える
	}
	return total
}

sum([]int{1, 2, 3})            // 6
sum([]float64{1.5, 2.5})       // 4.0
sum([]Gold{100, 250})          // 350(~int のおかげ)
```

> 🔍 **なぜ 10 年もなかったの? — ジェネリクスのジレンマ**
> Go は 2009 年の公開当初から「ジェネリクスがない」と批判され続けましたが、
> チームは急ぎませんでした。理由は有名な **ジェネリクスのジレンマ** です:
>
> - **C++ 方式**(型ごとにコードを複製): 実行は速いがコンパイルが遅く、バイナリが肥大化
> - **Java 方式**(型消去して 1 つの実装): コンパイルは速いが実行時に箱詰めコストが載る
>
> つまり「遅いプログラマ(書けなくて回り道)/遅いコンパイラ/遅い実行」の
> 三択で、Go はビルド速度を絶対に譲れなかった(第6章参照)ため、
> 納得できる設計が見つかるまで **10 年以上、interface{} と手書き複製でしのぐ**
> 選択をしました。最終的に採用されたのは両方式の中間(型の「形状」ごとに複製する
> GC shape stenciling)です。
> この歴史ゆえ、ネット上の 2022 年以前の Go コードには `interface{}` +
> 型アサーションによる「疑似ジェネリクス」が大量に残っています。
> 読めるようにはなっておき、自分では書かない——が現代の姿勢です。

## ジェネリック型 — 万能コンテナ本体

関数だけでなく、struct にも型パラメータを付けられます。

```go
// Queue は任意の型 T を先入れ先出しで運ぶ万能コンテナ
type Queue[T any] struct {
	items []T
}

func (q *Queue[T]) Push(x T) {
	q.items = append(q.items, x)
}

func (q *Queue[T]) Pop() (T, bool) {
	if len(q.items) == 0 {
		var zero T // 「T のゼロ値」を作る定番イディオム
		return zero, false
	}
	x := q.items[0]
	q.items = q.items[1:]
	return x, true
}
```

```go
q := Queue[string]{} // 型を指定してインスタンス化
q.Push("回復薬")
q.Push("鉄鉱石")
item, ok := q.Pop() // item は string 型。アサーション不要!
```

`Queue[string]` に int を Push するとコンパイルエラーです。
`interface{}` 時代は Pop のたびに型アサーションが必要で、間違いは実行時にしか
分かりませんでした。それが全部コンパイル時に移動した——これがジェネリクスの価値です。

## 標準ライブラリのジェネリクス

第4章で使った `slices` パッケージこそ、ジェネリクスの代表的な成果物です。

```go
slices.Contains(ids, "GX-0001")     // かつては型ごとに手書きしていた
slices.Max([]float64{3.2, 42.0})
slices.SortFunc(parcels, func(a, b Parcel) int {
	return cmp.Compare(a.Weight, b.Weight) // 重い順・軽い順も自在
})
maps.Keys(ledger) // maps パッケージも同様
```

## 使いすぎ注意 — インターフェースとの使い分け

ジェネリクスとインターフェース、どちらも「型を抽象化する」道具なので混同しがちです。

| 状況 | 使うもの |
|---|---|
| 呼び出しごとに **振る舞いが違う**(Truck と Drone) | インターフェース(第9章) |
| どの型でも **やることは同一**(合計、検索、コンテナ) | ジェネリクス |
| 単に「いろんな型を受けたい」だけ | まずインターフェースで足りないか考える |

Go チームの公式ガイドラインは「**まず interface で書け。型アサーションや
複製コードが増えてきたら generics**」です。Python から来ると
「とりあえず全部ジェネリックに」としたくなりますが、Go では具体型のままが基本、
抽象化は必要になった瞬間に、が流儀です。

## 🚇 完成コード: `express/day11/main.go`

```go
// Gopher Express — 万能設備への更新
package main

import (
	"cmp"
	"fmt"
	"slices"
)

type Number interface {
	~int | ~float64
}

// 型を問わない合計機
func sum[T Number](xs []T) T {
	var total T
	for _, x := range xs {
		total += x
	}
	return total
}

// 万能ベルトコンベア(FIFO)
type Queue[T any] struct{ items []T }

func (q *Queue[T]) Push(x T) { q.items = append(q.items, x) }
func (q *Queue[T]) Pop() (T, bool) {
	if len(q.items) == 0 {
		var zero T
		return zero, false
	}
	x := q.items[0]
	q.items = q.items[1:]
	return x, true
}

type Parcel struct {
	ID     string
	Weight float64
}

func main() {
	weights := []float64{3.2, 42.0, 0.4, 18.5}
	fmt.Printf("総重量: %.1fkg / 総運賃: %d\n",
		sum(weights), sum([]int{564, 1340, 508, 870}))

	// Parcel 専用に書き直すことなく、コンベアに載せる
	belt := Queue[Parcel]{}
	belt.Push(Parcel{ID: "GX-0001", Weight: 3.2})
	belt.Push(Parcel{ID: "GX-0002", Weight: 42.0})
	for {
		p, ok := belt.Pop()
		if !ok {
			break
		}
		fmt.Println("コンベアから搬出:", p.ID)
	}

	// slices + ジェネリクスで重量順に
	parcels := []Parcel{{"GX-1", 42.0}, {"GX-2", 0.4}, {"GX-3", 18.5}}
	slices.SortFunc(parcels, func(a, b Parcel) int {
		return cmp.Compare(a.Weight, b.Weight)
	})
	fmt.Println("軽い順:", parcels)
}
```

## 📝 今日の配達訓練(演習)

1. `func mapSlice[T, U any](xs []T, f func(T) U) []U` を実装してください。
   Python の `[f(x) for x in xs]` の Go 版です。`mapSlice(parcels, func(p Parcel) string { return p.ID })`
   で伝票番号一覧を作りましょう。
2. `Queue[T]` に `Len() int` と `Peek() (T, bool)` を追加してください。
   `var zero T` イディオムが必要になる場所はどこでしょう?
3. `sum` の制約から `~` を外し(`int | float64`)、第8章で作った `type Gold int` を
   渡すとどんなエラーになるか確認してください。`~` の役割が体感できます。

---

**🌳 ここから上級編です。** 設備は揃いました。しかし配達員は今も 1 匹ずつ
順番にトンネルへ潜っています。Go という言語の名前の由来にして最大の武器——
**goroutine と channel** で、配達を並行化します。Python の asyncio とは
まるで別物です。 → [第12章 配達員とトンネル](12_goroutines.md)
