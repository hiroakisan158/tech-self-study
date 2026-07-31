

# 第8章 荷物に振る舞いを — メソッドとレシーバ

## 🚇 今日のお話

カルテ(struct)はただのデータの束でした。今日は「運賃を自分で計算する」
「自分にチェックを入れる」といった **振る舞い(メソッド)** をカルテに教えます。

Python の class はデータとメソッドを 1 つの教室に同居させましたが、
Go では **struct を先に定義し、メソッドを外付けする** スタイルです。

## メソッド = レシーバ付きの関数

```go
type Parcel struct {
	ID     string
	Weight float64
}

// (p Parcel) がレシーバ。「Parcel に付けるメソッド」の宣言
func (p Parcel) Fare() int {
	return 500 + int(p.Weight*20)
}

p := Parcel{ID: "GX-0001", Weight: 3.2}
fmt.Println(p.Fare()) // 564
```

レシーバは Python の `self` に相当しますが、名前は自由です(慣習は型名の頭文字 1〜2 文字。
`self` や `this` はむしろ非推奨です)。

> 🐍 **Python との違い①: class ブロックがない**
> Python はクラス定義の中にメソッドを書きましたが、Go のメソッドは
> **struct 定義の外に、普通の関数として** 書きます(同じパッケージ内ならファイルも自由)。
> `p.Fare()` は実質 `Fare(p)` の糖衣で、「第 0 引数が特別な位置に来た関数」に過ぎません。
> Python も `Parcel.fare(p)` と書けば同じ形だったことを思い出すと腑に落ちます。

### 結局、普通の関数とメソッドは何が違うの?

見た目も動きもそっくりなので混乱しやすいですが、整理しておきます。

```go
// 普通の関数として書く場合
func Fare(p Parcel) int { return 500 + int(p.Weight*20) }
Fare(p)      // 呼び方: 関数名(引数)

// メソッドとして書く場合
func (p Parcel) Fare() int { return 500 + int(p.Weight*20) }
p.Fare()     // 呼び方: 値.メソッド名()
```

中でやっている計算は全く同じです。**「どの型に対する操作なのか」を、関数の
引数として渡すか(普通の関数)、`.` の前に置くか(メソッド)** の違いしか
ありません。名前の通り、メソッドは「レシーバ(受け取り役)が付いた関数」
というだけです。

とはいえ実用上は、次の2点でメソッドを選ぶ理由がはっきりします。

1. **`型.やること()` という自然な読み方ができる** — `Fare(p)`(「運賃(荷物)」と
   読める)より `p.Fare()`(「荷物の運賃」と読める)の方が、コードを読んだ
   ときにどの型の話をしているか一目で分かります。関連する操作を型ごとに
   まとめて整理できるのもメリットです
2. **インターフェースを満たせるのはメソッドだけ** — 第9章で学びますが、
   Go では「特定のメソッドを持っている型」を条件にして色々な型を同じ
   扱いにできます。この仕組みは**メソッド限定**で、普通の関数をいくら
   定義しても対象になりません。「型にどんな振る舞いがあるか」を後から
   問い合わせられるようにする、というのがメソッドの本当の存在意義です

今の時点では「メソッド = ある型専用の関数、`.`で呼べて型ごとに整理できる」
くらいの理解で十分です。インターフェースとの関係は第9章で実感を伴って
分かるようになります。

メソッドは自作の型ならなんにでも付けられます。struct 限定ではありません:

```go
type Gold int // int に別名を付けた「定義型」

func (g Gold) Format() string {
	return fmt.Sprintf("%d ゴールド", int(g))
}
```

Python で「int のサブクラス」を作るような場面が、Go では「定義型 + メソッド」になります。

## 値レシーバ vs ポインタレシーバ — 本章最大のポイント

前章の知識がそのまま効きます。レシーバも **引数と同じくコピーかポインタか** を選びます。

```go
// 値レシーバ: p はコピー。読むだけのメソッド向け
func (p Parcel) Fare() int { ... }

// ポインタレシーバ: p は原本。書き換えるメソッド向け
func (p *Parcel) MarkDelivered() {
	p.Delivered = true
}
```

```go
p := Parcel{ID: "GX-0001"}
p.MarkDelivered()        // (&p).MarkDelivered() と自動解釈してくれる
fmt.Println(p.Delivered) // true
```

`&` を書かなくても、**アドレスを取れる変数** ならコンパイラが自動で取ってくれます。

### ⚠️ 落とし穴①: 値レシーバで書き換えても消える

```go
func (p Parcel) MarkDelivered() { // 値レシーバにしてしまった
	p.Delivered = true // コピーに書いただけ。呼び出し後に蒸発
}
```

コンパイルは通り、テストも「エラーは出ない」ので、前章の事故のメソッド版として
静かに紛れ込みます。`go vet` でも捕まらないので、レシーバを決めるときに意識するしかありません。

### 使い分けの指針

1. **書き換えるメソッドが 1 つでもあるなら、全メソッドをポインタレシーバに統一**
   (混在させると「どれが原本に効くのか」を毎回考える羽目になります)
2. 完全に読み取り専用の小さな型(`time.Time` 型など)だけ値レシーバ
3. 迷ったらポインタレシーバ

### ⚠️ 落とし穴②: map の中の値にはメソッドで書き込めない

```go
ledger := map[string]Parcel{"GX-0001": {}}
ledger["GX-0001"].MarkDelivered() // ❌ コンパイルエラー
```

map の要素はアドレスが取れない(第5章の落とし穴③と同根)ため、
ポインタレシーバのメソッドを呼べません。`map[string]*Parcel` にするのが定石です。

> 🔍 **呼べないのはポインタレシーバだけ — 値レシーバなら普通に呼べる**
> `x.Method()` と書いたとき、`Method` がポインタレシーバなら Go は裏で
> `(&x).Method()` に読み替えます。この「こっそり `&` を付ける」処理には
> `x` のアドレスが取れることが前提で、map の要素はそれができないため
> コンパイルエラーになります。
>
> つまり止められているのは **アドレスが必要なポインタレシーバのメソッドだけ**
> です。読むだけの値レシーバなら、map の中の struct でも問題なく呼べます。
> ```go
> func (p Parcel) Summary() string { // 値レシーバ
> 	return p.ID
> }
>
> ledger["GX-0001"].Summary() // ✅ 通る(アドレス不要、コピーを渡すだけ)
> ```
> ただしメソッド内で `p` を書き換えても、それはコピーへの変更なので
> 呼び出し元の map には反映されません。「struct in map ではメソッドが
> 一切呼べない」のではなく、「書き込み(ポインタレシーバ)だけが呼べない」
> というのが正確な理解です。

## コンストラクタの慣習 — New○○

Go に `__init__` はありません。**「ゼロ値で使えるならコンストラクタ不要」** が第一原則で、
初期化が必要な型には `New○○` という普通の関数を用意します。

**なぜ必要なのか — まず失敗するパターンを見る**

`Book` を仮にこう定義したとします。

```go
type Book struct {
	parcels map[string]Parcel // 小文字 = パッケージ外からは触れない(第6章)
	nextID  int
}
```

もし利用側が `ledger.Book{}` のようにゼロ値で作ってしまうと、`parcels` は
**第5章で見た nil map** のままです。

```go
b := ledger.Book{}
b.Add("north", 3.2, true) // 💥 panic: assignment to entry in nil map
```

しかも `parcels` は小文字で非公開なので、パッケージの外からは
`ledger.Book{parcels: make(...)}` と書いて自分で初期化することも**できません**
(コンパイルエラーになります)。つまり利用側に残された選択肢は、壊れた
ゼロ値 `Book{}` を作るか、パッケージ側が用意した正しい作り方を使うかの
どちらかです。この「正しい作り方」を関数として用意するのが `New○○` です。

```go
func NewBook() *Book {
	return &Book{parcels: make(map[string]Parcel)} // ここで初めて make される
}
```

```go
b := ledger.NewBook() // 必ず初期化済みの Book が手に入る
b.Add("north", 3.2, true) // ✅ 安全
```

**なぜ `*Book`(ポインタ)を返すのか**
第7章で見た通り、`Book` を値で返すとコピーが渡ってしまい、後から
`Add` で台帳に追加しても呼び出し元の変数には反映されません(struct は
代入のたびに複写されるのでした)。「台帳の原本は1つで、みんなが同じ
原本を共有する」ために、`New○○` は基本的にポインタを返します。

**まとめると `New○○` がやっていることは3つ**

1. 非公開フィールド(map など)を正しく初期化する
2. ゼロ値では壊れる型でも、利用側には常に「使える状態」だけを渡す
3. ポインタを返すことで、以降のメソッド呼び出しが同じ原本に効くようにする

> 🔍 **`NewBook` はレシーバではない、呼ぶのは最初の1回だけ**
> `NewBook()` にはレシーバ(`(b *Book)` の部分)が付いていません。ただの
> 普通の関数です。使う流れはこうなります。
> ```go
> b := ledger.NewBook() // ① 新しい台帳が欲しい時に1回だけ呼ぶ
>
> b.Add("north", 3.2, true) // ② 同じ b に対してメソッドを何度でも呼ぶ
> b.Add("south", 1.0, false)
> p, ok := b.Find("GX-0001")
> ```
> `NewBook()` は「新しいインスタンスを1つ作りたい」タイミングでだけ呼びます
> (`Add` を呼ぶたびに呼び直すと、そのたびに別の空の台帳ができてしまいます)。
> `Add` や `Find` のようなレシーバ付きメソッドの方が、できた `b` に対して
> 何度も呼べる側です。

> 🔍 **`New○○` はメソッドを「呼べるようにする」わけではない**
> 誤解しやすい点として、メソッドが呼べるのは `New○○` を経由したからでは
> ありません。**メソッドは型に紐づいていて、その型の値であれば `New` を
> 使ったかどうかに関係なく常に呼び出せます**。
> ```go
> b := ledger.Book{}        // New を使わず、ゼロ値のまま作ってしまう
> b.Add("north", 3.2, true) // これもコンパイルは通る(=呼べてはいる)
>                            // 💥 だが nil map への書き込みで panic する
> ```
> `New○○` が保証しているのは「メソッドを呼ぶ権利」ではなく、
> **「メソッドを呼んでも安全に動く、正しく初期化された値」** です。
> 「`New` を使わないとメソッドが呼べない」のではなく、「`New` を使わないと
> 内部状態が壊れたままなので、メソッドを呼ぶと落ちることがある」という
> 理解の方が正確です。

こうして `New○○` + 非公開フィールドの組み合わせが、Python の `__init__` が
コンストラクタ引数を検証・整形して安全なインスタンスだけを作らせるのと
同じ役割を、Go 流(関数 + 命名規約 + 大文字小文字ルール)で実現しています。

## 継承はない。埋め込みがある

Go には継承がありません。型の再利用は **埋め込み(embedding)** で行います。

```go
type Vehicle struct {
	Name string
	Fuel int
}

func (v *Vehicle) Refuel() { v.Fuel = 100 }

type Truck struct {
	Vehicle  // フィールド名なしで型だけ書く = 埋め込み
	MaxLoad float64
}

t := Truck{Vehicle: Vehicle{Name: "1号車", Fuel: 20}, MaxLoad: 500}
t.Refuel()          // Vehicle のメソッドが「昇格」して直接呼べる
fmt.Println(t.Fuel) // 100(フィールドも昇格)
```

一見継承ですが、決定的な違いがあります: **Truck は Vehicle 型として扱えません**。
`var v Vehicle = t` はエラーです。埋め込みは「is-a(である)」ではなく
**「has-a(を持つ)+ 転送の自動化」** に過ぎません。

> 🔍 **なぜ継承を捨てたの?**
> 深い継承階層は「親のこの変更は子の何を壊すか」が追えなくなる問題
> (脆い基底クラス問題)を抱え、Java/C++ の世界でも「継承より合成
> (composition over inheritance)」が長く説かれてきました。Go はいっそ
> **継承を言語から削除し、合成(埋め込み)だけを残した** のです。
> オーバーライドもありません——Truck に同名メソッドを定義すれば外側が
> 勝ちますが、Vehicle 側のメソッドから「子の実装」が呼ばれること
> (Python の `super()` を軸にしたテンプレートメソッドパターン)は起きません。
> ポリモーフィズム(同じ扱いで違う動き)が欲しいときは、次章の
> **インターフェース** が担当します。「型の再利用 = 埋め込み」
> 「振る舞いの抽象化 = インターフェース」と、Python では class 1 つが
> 担っていた役割が Go では 2 つの道具に分かれています。

## 🚇 完成コード: `express/ledger/ledger.go` の進化

```go
package ledger

import "fmt"

type Parcel struct {
	ID        string
	Dest      string
	Weight    float64
	Delivered bool
}

// Fare は運賃を計算する(読み取り専用でも、書き換え系メソッドを持つ型なので
// ポインタレシーバに統一)。
func (p *Parcel) Fare() int {
	return 500 + int(p.Weight*20)
}

func (p *Parcel) MarkDelivered() {
	p.Delivered = true
}

// String は fmt での表示形式を定義する(Python の __str__ 相当。第9章で種明かし)
func (p *Parcel) String() string {
	status := "🚚 配達中"
	if p.Delivered {
		status = "✅ 配達済"
	}
	return fmt.Sprintf("[%s] %s 行き %.1fkg %s", p.ID, p.Dest, p.Weight, status)
}

type Book struct {
	parcels map[string]*Parcel // ポインタの map: 原本を 1 つに保つ
	nextID  int
}

func NewBook() *Book {
	return &Book{parcels: make(map[string]*Parcel)}
}

func (b *Book) Add(dest string, weight float64) *Parcel {
	b.nextID++
	p := &Parcel{ID: fmt.Sprintf("GX-%04d", b.nextID), Dest: dest, Weight: weight}
	b.parcels[p.ID] = p
	return p
}

func (b *Book) Find(id string) (*Parcel, bool) {
	p, ok := b.parcels[id]
	return p, ok
}
```

```go
// main.go
package main

import (
	"fmt"

	"gopher.example/express/ledger"
)

func main() {
	book := ledger.NewBook()
	p := book.Add("north", 3.2)
	fmt.Println(p) // String() が自動で使われる

	p.MarkDelivered() // 原本にチェック。台帳側の Find でも反映済み
	if found, ok := book.Find(p.ID); ok {
		fmt.Println(found) // ✅ 配達済
	}
}
```

## 📝 今日の配達訓練(演習)

1. 落とし穴①を再現してください: `MarkDelivered` を値レシーバに変えて、
   チェックが蒸発することを確認し、ポインタレシーバに戻してください。
2. `type Km float64` を定義し、`func (k Km) ToFuelCost() Gold` のような
   メソッドを付けてください。「単位を型にする」と `fare + distance` のような
   単位違いの計算がコンパイルエラーになる利点を確認しましょう。
3. `Vehicle` を埋め込んだ `Drone` を作り、`Refuel` が昇格することと、
   `var v Vehicle = drone` がコンパイルエラーになることの両方を確認してください。

---

トラック、船、ドローン——車両が増えてきました。「積んで、走れるなら、
車種は問わない」と言いたい。Python ではダックタイピングで暗黙にやっていたことを、
Go は **インターフェース** で型安全にやります。Go 設計の白眉です。
→ [第9章 どんな車両でも走れる免許](09_interfaces.md)
