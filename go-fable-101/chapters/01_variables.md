# 第1章 伝票と帳簿 — 変数と型

## 🚇 今日のお話

あなたは今日から地下トンネル運送会社「Gopher Express」の新米所長です。
まずやることは 2 つ。**会社の看板を掲げる** ことと、**帳簿に今日の売上を記録する** ことです。

Go のプログラムは、Python と違って「いきなり 1 行目から実行」はできません。
必ず **パッケージ宣言** と **main 関数** という決まった器から始まります。

```go
package main

import "fmt"

func main() {
	companyName := "Gopher Express"
	fmt.Println("🚇", companyName, "開業!")
}
```

```bash
go run main.go
```

> 🐍 **Python との違い①: 実行の入り口が決まっている**
> Python はファイルの上から順に実行されますが、Go は `package main` の中の
> `func main()` だけが入り口です。スクリプトというより「アプリを組み立てて起動する」
> 感覚に近くなります。

## 変数の宣言 — 2 つの書き方

Go には変数の宣言方法が主に 2 つあります。

```go
// ① var: 型を明示する(またはゼロ値で初期化する)とき
var revenue int = 0
var companyName string = "Gopher Express"

// ② := : 型推論つきの短縮宣言(関数の中でだけ使える)
parcels := 12          // int と推論される
fuelCost := 34.5       // float64 と推論される
isOpen := true         // bool と推論される
```

実際の Go コードでは、関数内は `:=`、パッケージレベル(関数の外)や
「ゼロ値から始めたい」ときは `var`、と使い分けるのが慣習です。

> 🐍 **Python との違い②: 静的型付け**
> Python では `x = 1` のあとに `x = "たくさん"` と書けましたが、Go では
> `parcels := 12` した変数に文字列を代入するとコンパイルエラーです。
> 変数は「値に貼るラベル」ではなく **「型が決まった専用の入れ物」** です。
> Python 第13章で学んだ型ヒントが、Go では言語の必須機能になったと思ってください。
> mypy のような別ツールは不要で、コンパイラ自身が常に型を検査します。

## ゼロ値 — Go に「未初期化」はない

Python では値を入れる前の変数は存在しませんが、Go では **宣言しただけで必ず初期値が入ります**。
これを **ゼロ値** と呼びます。

```go
var count int       // 0
var price float64   // 0.0
var name string     // "" (空文字列)
var done bool       // false

fmt.Println(count, price, name, done) // 0 0  false
```

| 型 | ゼロ値 |
|---|---|
| 数値型 (`int`, `float64` など) | `0` |
| `string` | `""` |
| `bool` | `false` |
| ポインタ、スライス、map、channel、インターフェース | `nil` |

> 🔍 **なぜそうなっているの? — ゼロ値の思想**
> C 言語では未初期化変数に「メモリに残っていたゴミ」が入っており、これが
> 長年バグの温床でした。Go の設計者たち(C の開発に関わった Ken Thompson、
> Rob Pike ら)はこれを反省し、**「宣言された変数は必ず使える状態にする」** と
> 決めました。さらに Go の標準ライブラリは「ゼロ値のまま使える」設計
> (例: `var sb strings.Builder` はそのまま使える)を推奨しており、
> ゼロ値は単なる初期値ではなく **設計の道具** になっています。
> Python の `None` は「値がない」の表明ですが、Go のゼロ値は「もう使える」の表明です。

## 使わない変数はコンパイルエラー

```go
func main() {
	unused := 42 // ← これだけでコンパイルエラー: declared and not used
}
```

> 🔍 **なぜそうなっているの? — 「警告」が存在しない言語**
> 多くの言語では未使用変数は「警告」止まりですが、警告は無視され続けて
> ノイズになりがちです。Google の巨大コードベースで「警告が数千件あるビルドログ」に
> 悩まされた経験から、Go は **警告という概念自体を捨て、エラーか適法かの二択** に
> しました。未使用 import も同様にエラーです。デバッグ中に一時的に消したいときは
> ブランク識別子 `_ = unused` で「意図的に捨てる」と明示します。

## 基本の型たち

| 型 | 例 | 帳簿での用途 |
|---|---|---|
| `int` | `100`, `-5` | 荷物の個数、売上(整数) |
| `float64` | `34.5` | 燃料費、割引率 |
| `string` | `"回復薬 30 箱"` | 品名、宛先 |
| `bool` | `true` / `false` | 配達済みフラグ |
| `byte` (= `uint8`) | `'A'` | 生バイトデータ |
| `rune` (= `int32`) | `'荷'` | Unicode 1 文字 |

### 暗黙の型変換はない

```go
var count int = 3
var rate float64 = 0.1

// total := count * rate        // コンパイルエラー! int と float64 は混ぜられない
total := float64(count) * rate  // 明示的に変換する
```

> 🐍 **Python との違い③: `3 * 0.1` すら書けない**
> Python は `int` と `float` を自由に混ぜられますが、Go は **同じ数値でも型が違えば
> 明示変換が必要** です。面倒に感じますが、C 言語の暗黙変換(符号の有無や桁あふれで
> 静かに値が壊れる)による事故の歴史への反省です。「変換が起きる場所がコード上に
> 必ず見える」ことを Go は選びました。
> なお `total := 3 * 0.1` のように **リテラル同士** なら書けます。Go の数値リテラルは
> 「型が未確定の定数」で、代入先に合わせて型が決まるからです(後述の定数も同じ仕組み)。

### 文字列と rune — 「文字数」の罠

```go
label := "荷物"
fmt.Println(len(label))        // 6 ← バイト数!(UTF-8 で漢字は 3 バイト)
fmt.Println(len([]rune(label))) // 2 ← 文字数が欲しいときはこう
```

Go の `string` は **UTF-8 のバイト列** です。Python 3 の `str` が「文字の列」だったのと
対照的で、`len` はバイト数を返します。日本語を扱うときは要注意です。

## 定数と iota — 料金表を作る

```go
const baseFare = 500 // 基本料金(変更不可)

// iota: 0, 1, 2... と自動連番される定数生成器
const (
	SizeS = iota // 0
	SizeM        // 1
	SizeL        // 2
	SizeXL       // 3
)
```

`const` はコンパイル時に値が決まる真の定数です。Python の「大文字で書く紳士協定」と違い、
代入しようとするとコンパイルエラーになります。

## fmt.Printf — 伝票の印字

f-string の代わりに、Go では `fmt` パッケージの **書式指定(verb)** を使います。

```go
name := "回復薬"
count := 30
weight := 12.345

fmt.Printf("品名: %s / 個数: %d 箱 / 重量: %.1f kg\n", name, count, weight)
// 品名: 回復薬 / 個数: 30 箱 / 重量: 12.3 kg

fmt.Printf("%v\n", count)  // %v: なんでも良い感じに表示(迷ったらこれ)
fmt.Printf("%T\n", weight) // %T: 型を表示 → float64
line := fmt.Sprintf("%s x%d", name, count) // 表示せず文字列を作る
```

| verb | 意味 | Python でいうと |
|---|---|---|
| `%v` | デフォルト表示 | `{x}` |
| `%d` | 整数 | `{x:d}` |
| `%.1f` | 小数第 1 位まで | `{x:.1f}` |
| `%s` | 文字列 | `{x}` |
| `%q` | 引用符つき文字列 | `{x!r}` に近い |
| `%T` | 値の型 | `type(x)` |
| `%+v` | struct をフィールド名つきで | — |

> 🐍 f-string(`f"{name} x{count}"`)に慣れていると最初は退屈ですが、
> `%+v` や `%#v`(Go の構文として表示)など **デバッグ特化の verb** は Go の方が強力です。

## 🚇 完成コード: `express/day1/main.go`

```go
// Gopher Express — 入社 1 日目
package main

import "fmt"

const baseFare = 500 // 基本料金(ゴールド)

func main() {
	companyName := "Gopher Express"
	var revenue int // ゼロ値 0 から開始

	fmt.Printf("🚇 %s 営業開始!\n", companyName)

	// 最初の配達依頼: 回復薬 30 箱、重量 12.3kg
	item := "回復薬"
	count := 30
	weight := 12.3
	fare := baseFare + int(weight*20) // 重量 1kg ごとに 20 ゴールド加算

	fmt.Printf("伝票: %s x%d(%.1fkg)→ 運賃 %d ゴールド\n", item, count, weight, fare)

	revenue += fare
	fmt.Printf("本日の売上: %d ゴールド\n", revenue)
}
```

```bash
go run express/day1/main.go
```

## 📝 今日の配達訓練(演習)

1. `const expressFee = 300`(速達料金)を追加し、速達の場合の運賃も印字してください。
2. `var discount float64` を宣言してそのまま表示し、ゼロ値を確認してください。
   その後 `0.2`(2 割引)を代入し、割引後運賃を `%.0f` で表示してください。
   `fare * discount` と書くとどんなコンパイルエラーが出るかも観察しましょう。
3. `fmt.Println(len("トンネル"))` の結果を予想してから実行してください。
   `for i, r := range "トンネル"` で 1 文字ずつ表示するとどうなるかも試してみましょう
   (range は次章で正式に学びます)。

---

次章、荷物が続々と届き始めます。行き先ごとに仕分けるには **条件分岐と繰り返し** が
必要です。ただし Go にはループが 1 種類しかありません…? → [第2章 配達ルートの判断](02_control_flow.md)
