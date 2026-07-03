# 第5章 宛先台帳と荷物カルテ — map と struct

## 🚇 今日のお話

今日は事務仕事です。「伝票番号 → 荷物」を一発で引ける **宛先台帳(map)** と、
1 つの荷物の情報をひとまとめにする **荷物カルテ(struct)** を整備します。

Python でいえば `dict` と `class`(または `dataclass`)に当たりますが、
どちらも Go らしい違いと落とし穴があります。

## map — 宛先台帳

```go
// map[キーの型]値の型
fares := map[string]int{
	"north": 500,
	"south": 700,
}

fares["west"] = 650          // 追加・更新
delete(fares, "south")       // 削除
fmt.Println(len(fares))      // 2
```

### カンマ ok イディオム — 「ない」の確認

存在しないキーを引いても Python のような `KeyError` は起きず、**値のゼロ値** が返ります。

```go
fare := fares["east"]     // 0 が返る! 「運賃 0 ゴールド」と区別できない…
fare, ok := fares["east"] // 第 2 戻り値で存在を確認できる
if !ok {
	fmt.Println("未開通の行き先です")
}
```

> 🐍 **Python との違い①: KeyError がない世界**
> Python の `d["x"]` は例外、`d.get("x")` は `None` でした。Go は常に
> `d.get("x", ゼロ値)` 相当が返り、区別したければ `, ok` を受け取ります。
> **「ゼロ値が正当な値になりうる map」では `, ok` を省略してはいけません**。
> `fares["east"]` が 0 を返したとき、それは「無料」なのか「未登録」なのか——
> ここを曖昧にしたバグは型検査でも捕まりません。

### ⚠️ 落とし穴①: nil map には書き込めない

```go
var book map[string]int   // ゼロ値は nil
fmt.Println(book["x"])    // 0 — 読むのは OK
book["x"] = 1             // 💥 panic: assignment to entry in nil map
book = make(map[string]int) // 書き込む前に make(またはリテラル)で初期化
```

nil スライスへの `append` は OK だったのに、nil map への書き込みは panic——
非対称ですが、「append は新しい実体を返せるが、map への代入は返す先がない」
と考えると仕組み上の必然です。**map は使う前に必ず `make` する**、と覚えてください。

### ⚠️ 落とし穴②: 反復順序は毎回ランダム

```go
for dest, fare := range fares {
	fmt.Println(dest, fare) // 実行するたびに順番が変わる!
}
```

> 🔍 **なぜそうなっているの? — わざとランダムにしている**
> Go の map の反復順序は「不定」ではなく、ランタイムが **意図的に毎回シャッフル**
> しています。初期の Go で、たまたま安定していた順序にユーザーのコードが依存し、
> ランタイム改善で順序が変わるたびにプログラムが壊れる事故が起きました。
> そこで開発チームは「仕様外の挙動には最初から依存できなくしてしまえ」と、
> 反復開始位置を乱数で選ぶようにしたのです(Hyrum の法則:
> 「観測可能な挙動はすべて誰かに依存される」への実力行使)。
> 順序が必要なら、キーを取り出して `slices.Sort` してから回します。
>
> 🐍 Python の dict が 3.7 から **挿入順を保証** したのと真逆の決断です。
> Python は「便利だから仕様に昇格」、Go は「依存されないよう破壊」——
> 両言語の性格がいちばんよく出ている違いかもしれません。

```go
import "maps" // Go 1.23+
for _, dest := range slices.Sorted(maps.Keys(fares)) {
	fmt.Println(dest, fares[dest]) // 常にキー昇順
}
```

## struct — 荷物カルテ

複数の値をひとまとめにする型を自分で定義します。

```go
type Parcel struct {
	ID     string
	Dest   string
	Weight float64
	Fragile bool
}

p := Parcel{ID: "GX-0001", Dest: "north", Weight: 3.2, Fragile: true}
fmt.Println(p.Dest)   // north
p.Weight = 4.0        // フィールドの更新
fmt.Printf("%+v\n", p) // {ID:GX-0001 Dest:north Weight:4 Fragile:true}
```

- フィールド名を省いた `Parcel{"GX-0001", "north", 3.2, true}` も書けますが、
  フィールド追加で壊れるため **名前付きで書くのが慣習** です
- 指定しなかったフィールドは **ゼロ値** になります(コンストラクタ必須の Python と違い、
  `Parcel{}` はそれだけで完全に合法な「空のカルテ」です)

> 🐍 **Python との違い②: class ではなく「ただのデータの束」**
> Go の struct は `@dataclass` にいちばん近い存在ですが、`__init__` も継承も
> ありません。メソッドは後から外付けします(第8章)。Go には class という
> 概念自体がなく、**「データ(struct)」と「振る舞い(メソッド)」と
> 「契約(インターフェース)」を別々の部品として組み立てます**。この分解が
> Go のオブジェクト指向の全体像で、第7〜9章で 1 つずつ学びます。

### struct は値 — 代入はまるごとコピー

```go
a := Parcel{ID: "GX-0001", Weight: 3.2}
b := a          // 全フィールドのコピー!
b.Weight = 99
fmt.Println(a.Weight) // 3.2 — a は無傷
```

Python でオブジェクトを代入すると「同じ実体への参照が 2 つ」でしたが、
Go の struct 代入は **カルテの複写** です。関数に渡すときも同じくコピーが渡ります。
「元を書き換えたいのに書き換わらない」問題と、その解決策(ポインタ)は
第7章のメインテーマです。

### 比較とマップのキー

フィールドがすべて比較可能なら、struct は `==` で比較でき、**map のキーにもなれます**。

```go
type Route struct{ From, To string }
traffic := map[Route]int{}
traffic[Route{"north", "south"}]++ // タプルをキーにする Python の dict と同じ発想
```

Python では `__eq__` を書くか `dataclass(frozen=True)` が必要だった芸当が、
Go では標準で手に入ります(スライスや map を含む struct は比較不可になる点に注意)。

## map とスライスの中の struct — 落とし穴③

```go
parcels := map[string]Parcel{"GX-0001": {Weight: 3.2}}
parcels["GX-0001"].Weight = 4.0 // ❌ コンパイルエラー!
```

map から取り出した struct は **コピー** なので、その場でフィールドに代入できません
(代入してもコピーが捨てられるだけなので、コンパイラが禁止しています)。

```go
p := parcels["GX-0001"] // ① 取り出して
p.Weight = 4.0          // ② 直して
parcels["GX-0001"] = p  // ③ 戻す
```

または最初から `map[string]*Parcel`(ポインタの map)にします。第7章で再訪します。

## 🚇 完成コード: `express/day5/main.go`

```go
// Gopher Express — 台帳とカルテの整備
package main

import (
	"fmt"
	"maps"
	"slices"
)

type Parcel struct {
	ID      string
	Dest    string
	Weight  float64
	Fragile bool
}

func main() {
	// 伝票番号 → 荷物カルテ の台帳
	ledger := make(map[string]Parcel)
	ledger["GX-0001"] = Parcel{ID: "GX-0001", Dest: "north", Weight: 3.2, Fragile: true}
	ledger["GX-0002"] = Parcel{ID: "GX-0002", Dest: "south", Weight: 42.0}
	ledger["GX-0003"] = Parcel{ID: "GX-0003", Dest: "north", Weight: 0.4}

	// 問い合わせ対応
	if p, ok := ledger["GX-0002"]; ok {
		fmt.Printf("照会: %+v\n", p)
	}

	// 日報は伝票番号順で(map の順序は当てにしない!)
	fmt.Println("--- 本日の台帳 ---")
	for _, id := range slices.Sorted(maps.Keys(ledger)) {
		p := ledger[id]
		mark := ""
		if p.Fragile {
			mark = " ⚠️割れ物"
		}
		fmt.Printf("%s: %s 行き %.1fkg%s\n", id, p.Dest, p.Weight, mark)
	}
}
```

## 📝 今日の配達訓練(演習)

1. 行き先ごとの荷物数を数える `map[string]int` を作ってください。
   存在しないキーへの `counts[dest]++` がなぜ初期化なしで動くのか、
   ゼロ値の観点から説明してみましょう(これは Python の `defaultdict(int)` 相当が
   標準になっている、数少ない「Go の方が短く書ける」例です)。
2. 落とし穴①を再現してください: `var m map[string]int` に書き込んで panic を見た後、
   `make` で直してください。
3. `Route{From, To string}` をキーに、通行量を記録する map を作り、
   同じ Route で 2 回インクリメントすると合算されることを確かめてください。

---

台帳もカルテも整いましたが、コードが 1 ファイルに詰め込まれて限界です。
台帳係・料金係・受付係を **別々の部屋(パッケージ)** に分けましょう。
Go の「大文字で始まる名前だけが公開される」ルールも登場します。
→ [第6章 営業所を増やす](06_packages.md)
