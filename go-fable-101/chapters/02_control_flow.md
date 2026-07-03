# 第2章 配達ルートの判断 — if / for / switch

## 🚇 今日のお話

Gopher Express に荷物が続々と届き始めました。所長の仕事は **仕分け** です。
「北トンネル行き」「南トンネル行き」「割れ物注意」…… 荷物ごとに判断し、
全部の荷物を処理するまで繰り返す。つまり **条件分岐** と **ループ** の出番です。

## if — 括弧なし、中括弧必須

```go
weight := 25.0

if weight > 20 {
	fmt.Println("大型荷物: 貨物トンネルへ")
} else if weight > 5 {
	fmt.Println("中型荷物: 通常トンネルへ")
} else {
	fmt.Println("小型荷物: 速達トンネルへ")
}
```

- 条件に `( )` は付けません(付けても動きますが `gofmt` が外します)
- 本体の `{ }` は 1 行でも省略できません
- `else` は **`}` と同じ行に** 書かなければコンパイルエラーです

> 🔍 **なぜそうなっているの? — `else` の位置まで強制される理由**
> Go はセミコロンを書かない言語ですが、実はコンパイラが行末に自動挿入しています。
> `}` の後に改行するとそこにセミコロンが入り、`else` が文法的に迷子になるのです。
> この「自動セミコロン挿入」ルールの副産物として、Go では **中括弧のスタイル論争が
> 起きません**。`gofmt` と合わせて「書き方の議論を言語仕様で終わらせる」のが Go 流です。
> Python がインデントで論争を終わらせたのと、目的は同じで手段が違います。

### if の初期化文 — Go らしいイディオム

条件の前に `;` 区切りで短い文を書けます。**変数のスコープを if の中に閉じ込める** ための機能で、
後の章のエラー処理で多用します。

```go
if fare := calcFare(weight); fare > 1000 {
	fmt.Println("高額配送:", fare)
} // fare はここではもう使えない
```

> 🐍 **Python との違い①: 三項演算子がない**
> `x = a if cond else b` に相当する構文は Go にありません。素直に if 文で書きます。
> 「短く書ける」より「読み方が 1 通り」を優先する Go の思想です。同じ理由で
> `map()` や内包表記もなく、だいたいのことは for で書きます。

## for — ループはこれ 1 つだけ

Go には `while` がありません。**ループは `for` だけ** で、書き方で使い分けます。

```go
// ① C スタイル: 回数が決まっているとき
for i := 0; i < 5; i++ {
	fmt.Println("荷物", i, "を積み込み")
}

// ② while スタイル: 条件だけ書く
fuel := 100
for fuel > 0 {
	fuel -= 30
}

// ③ 無限ループ: 条件も書かない(break で抜ける)
for {
	fmt.Println("巡回中...")
	break
}

// ④ range: コレクションを回す(Python の for-in に相当)
items := []string{"回復薬", "羊皮紙", "鉄鉱石"}
for i, item := range items {
	fmt.Printf("%d 番目: %s\n", i, item)
}
```

> 🐍 **Python との違い②: `enumerate` が標準動作**
> `range` は **インデックスと値のペア** を返します。Python の
> `for i, item in enumerate(items)` が最初から標準形だと思ってください。
> 値だけ欲しいときは `for _, item := range items`、
> インデックスだけなら `for i := range items` と書きます。
> `_`(ブランク識別子)は「受け取るが捨てる」の明示で、第1章の未使用変数エラーを
> 回避する公式の方法です。

### ⚠️ 落とし穴: range の値はコピー

```go
type parcel struct{ delivered bool }
parcels := []parcel{{false}, {false}}

for _, p := range parcels {
	p.delivered = true // ← コピーに書いている!元は変わらない
}
fmt.Println(parcels) // [{false} {false}]

for i := range parcels {
	parcels[i].delivered = true // ✅ インデックス経由で本体を書き換える
}
```

`range` が渡す変数は要素の **コピー** です。Python では「すべてが参照」なので
ループ変数経由で中身を変更できましたが、Go は **値のコピーが基本** です
(この本質は第7章ポインタで深掘りします)。

> 🔍 **なぜそうなっているの? — ループ変数をめぐる Go 史上最大級の変更**
> かつての Go では、ループ変数はループ全体で **1 個の変数を使い回して** いました。
> そのためループ内でクロージャや goroutine を作ると「全部が最後の値を見る」という
> 有名なバグ(`for i := ...` の `i` が全部 3 になる等)が量産されました。
> あまりに事故が多く、**Go 1.22(2024年)で「反復ごとに新しい変数を作る」に仕様変更**
> されました。互換性を何より重んじる Go が仕様を破ってまで直した、史上まれな例です。
> ネット上の古い記事には「ループ変数をコピーする回避策(`i := i`)」が大量に
> 残っているので、読むときは Go のバージョンに注意してください。

## switch — break を書かない switch

```go
dest := "north"

switch dest {
case "north":
	fmt.Println("北トンネル: 第 1 ゲートへ")
case "south", "west": // 複数の値をまとめられる
	fmt.Println("南西トンネル: 第 2 ゲートへ")
default:
	fmt.Println("行き先不明: 保留棚へ")
}
```

C や JavaScript と違い、**case は自動で終わります**。`break` は不要です。

> 🔍 **なぜそうなっているの? — fallthrough の逆転**
> C の switch は「break を書き忘れると次の case に落ちる」仕様で、これが
> 数十年にわたりバグを生み続けました(Apple の SSL 通信の脆弱性 “goto fail” も
> この系統)。Go は **デフォルトを安全側に倒し**、あえて落としたいときだけ
> `fallthrough` と明示させる設計にしました。「危険な方を長く書かせる」のは
> Go 全体に通じる設計原則です。

### 条件 switch — else-if の梯子をきれいに

`switch` の後に値を書かないと、**各 case が条件式** になります。

```go
switch {
case weight > 20:
	fmt.Println("大型")
case weight > 5:
	fmt.Println("中型")
default:
	fmt.Println("小型")
}
```

長い `else if` の連鎖はこの形で書くのが Go の慣習です。
Python 3.10 の `match` 文は「構造のパターンマッチ」でしたが、Go の switch は
もっと素朴な「きれいな if の連鎖」と捉えてください
(型による分岐 `switch v := x.(type)` は第9章で登場します)。

## 🚇 完成コード: `express/day2/main.go`

```go
// Gopher Express — 仕分け業務
package main

import "fmt"

func main() {
	// 品名と重量(kg)の伝票データ
	items := []string{"回復薬", "鉄鉱石", "羊皮紙", "ドラゴンの鱗"}
	weights := []float64{3.2, 42.0, 0.4, 18.5}

	revenue := 0
	for i, item := range items {
		w := weights[i]

		var gate string
		switch {
		case w > 20:
			gate = "貨物トンネル"
		case w > 5:
			gate = "通常トンネル"
		default:
			gate = "速達トンネル"
		}

		fare := 500 + int(w*20)
		revenue += fare
		fmt.Printf("📦 %-8s %5.1fkg → %s(運賃 %d)\n", item, w, gate, fare)
	}

	fmt.Printf("本日 %d 件配達、売上 %d ゴールド\n", len(items), revenue)
}
```

## 📝 今日の配達訓練(演習)

1. FizzBuzz 配達版: 1〜30 番の荷物のうち、3 の倍数は「割れ物注意」、5 の倍数は
   「天地無用」、両方なら「取扱厳重注意」と印字してください(`%` は Go でも剰余です)。
2. `for` の条件だけの形(while 相当)を使い、燃料 100 から 1 回の走行で 17 ずつ減らし、
   燃料が尽きるまで何回走れるか数えてください。
3. `dest := "east"` として switch に `fallthrough` を入れ、動きがどう変わるか実験してください。

---

同じ仕分けコードを毎日コピペするのは所長の仕事ではありません。作業を **関数** に
まとめて仕分け係に任せましょう。Go の関数は「値を 2 つ返す」のが当たり前…?
→ [第3章 仕分け係を雇う](03_functions.md)
