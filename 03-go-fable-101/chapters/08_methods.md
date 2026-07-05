

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

## コンストラクタの慣習 — New○○

Go に `__init__` はありません。**「ゼロ値で使えるならコンストラクタ不要」** が第一原則で、
初期化が必要な型には `New○○` という普通の関数を用意します。

```go
func NewBook() *Book {
	return &Book{parcels: make(map[string]*Parcel)}
}
```

map を含む型はゼロ値では使えない(nil map!)ので、`NewBook` 経由の生成を
公開 API とし、struct のフィールドを小文字にして直接生成を防ぎます。
第6章の「小文字 = 非公開」がカプセル化の道具として効いてくる場面です。

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
