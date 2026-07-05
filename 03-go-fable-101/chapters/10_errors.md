# 第10章 事故報告書の書き方 — エラー処理の設計

## 🚇 今日のお話

配達には事故がつきものです。宛先不明、積載オーバー、トンネル崩落。
これまで `errors.New("...")` で雑な報告書を書いてきましたが、今日は
**「誰が読んでも原因をたどれて、機械でも判別できる報告書」** の書き方を学びます。

第3章で「Go は例外を投げず、エラーを返す」ことは学びました。この章はその応用編
——エラーという **値** をどう設計するか、です。

## error はただのインターフェース

```go
type error interface {
	Error() string
}
```

前章の知識でこれが読めるはずです: **`Error() string` を持つ型は何でもエラーになれる**。
`errors.New` は「文字列を 1 つ持つだけの最小のエラー型」を返す関数に過ぎません。

## エラー設計 3 段階

### ① 番兵エラー — 「この失敗」を名前で公開する

```go
package ledger

import "errors"

var ErrNotFound = errors.New("荷物が見つかりません")

func (b *Book) Find(id string) (*Parcel, error) {
	p, ok := b.parcels[id]
	if !ok {
		return nil, ErrNotFound
	}
	return p, nil
}
```

パッケージ変数として公開されたエラーを **番兵エラー(sentinel error)** と呼びます。
慣習として名前は `Err` で始めます。呼び出し側は「どの失敗か」を判別できます。

> 🐍 Python の `except FileNotFoundError:` のように **例外の型** で分岐していたことを、
> Go では **エラーの値** で行うイメージです。標準ライブラリの `io.EOF` や
> `sql.ErrNoRows` がこのパターンです。

### ② カスタムエラー型 — 報告書に添付資料を付ける

メッセージだけでなくデータを運びたいときは、struct でエラー型を作ります。

```go
type OverloadError struct {
	Weight   float64
	Capacity float64
}

func (e *OverloadError) Error() string {
	return fmt.Sprintf("積載オーバー: %.1fkg(上限 %.1fkg)", e.Weight, e.Capacity)
}
```

### ③ ラップ — 報告書に経緯を積み重ねる

エラーは呼び出し階層を上へ運ばれるうちに文脈を失います。
`fmt.Errorf` の **`%w`** で「原因エラーを中に包んだまま」文脈を足せます。

```go
func loadManifest(path string) error {
	f, err := os.Open(path)
	if err != nil {
		return fmt.Errorf("積荷目録 %s の読み込み: %w", path, err)
	}
	defer f.Close()
	// ...
	return nil
}
// 最終的なメッセージ:
// 「配達準備: 積荷目録 north.txt の読み込み: open north.txt: no such file or directory」
```

各層が一言ずつ文脈を足すと、最上流には **事故の全経緯が 1 行に連なった報告書** が届きます。
Python のトレースバック(自動で全部付く)と違い、Go は **各層が手で文脈を足す** 文化です。

## errors.Is / errors.As — 包み紙ごしに中身を調べる

ラップされたエラーは `==` や型アサーションでは判別できません。専用の道具を使います。

```go
// Is: 包みの中に「この番兵」がいるか?
if errors.Is(err, ledger.ErrNotFound) {
	fmt.Println("伝票番号をお確かめください")
}

// As: 包みの中に「この型」がいるか? いたら取り出す
var oe *OverloadError
if errors.As(err, &oe) {
	fmt.Printf("あと %.1fkg 減らしてください\n", oe.Weight-oe.Capacity)
}
```

| 判別したいもの | 使う道具 |
|---|---|
| 特定の番兵エラー | `errors.Is(err, ErrX)` |
| 特定のエラー型(データも欲しい) | `errors.As(err, &target)` |
| 単なる `err != nil` 以上の情報が不要 | 何もしない(それで十分な場面が大半) |

> 🐍 **Python との違い①: except 階層 vs Is/As**
> Python は例外クラスの **継承階層** で分類しました(`OSError` を捕まえれば
> `FileNotFoundError` も捕まる)。Go に継承はないので、分類は
> 「ラップの連鎖を `Is`/`As` で掘る」ことで実現します。
> `raise X from Y`(原因の連鎖)に相当するのが `%w` によるラップです。

## panic と recover — 本当の緊急事態だけ

Go にも大域脱出はあります。**panic** です。ただし用途は例外とは違います。

```go
func mustLoadConfig() Config {
	cfg, err := load("config.toml")
	if err != nil {
		panic("設定ファイルが壊れています: " + err.Error()) // 起動不能 = 続行無意味
	}
	return cfg
}
```

- **error**: 起こりうる失敗(宛先不明、ネットワーク断)。呼び出し側が対処する
- **panic**: プログラミングのバグや続行不可能な状態(配列の範囲外、nil デリファレンス、
  起動時の設定破損)。基本、対処せずクラッシュさせる

`recover` を使うと defer の中で panic を捕まえられますが、使いどころは
「サーバーが 1 リクエストのバグで全体を道連れにしないための最終防衛線」など、
フレームワーク的な場面にほぼ限られます。

```go
func safeDeliver(c Carrier, dest string) {
	defer func() {
		if r := recover(); r != nil {
			fmt.Println("🚨 配達員がパニック! 回収しました:", r)
		}
	}()
	c.Deliver(dest) // 中でバグって panic しても営業所は止まらない
}
```

> 🔍 **なぜ「例外を捨てた」のに panic はあるの?**
> Go の主張は「例外が悪」ではなく「**普通の失敗を例外で流すのが悪**」です。
> ファイルがない・入力が不正——これらは異常ではなく日常であり、
> 戻り値として本流のコードで扱うべきだ、と。一方、配列の範囲外アクセスのような
> 「プログラマの誤り」まで戻り値にすると全コードがエラー処理で埋まるため、
> そこは panic に残しました。
> **判定基準は「呼び出し側に回復の手立てがあるか」** です。あるなら error、
> ないなら panic。Python が `LBYL より EAFP`(とりあえずやって except)を
> 好んだのと対照的に、Go は「失敗しうる操作の結果を毎回その場で見る」文化です。
> `if err != nil` の羅列は冗長ですが、**エラー処理が本流ロジックと同じ画面に
> 常に見えている** ——これが Go が意図的に払っているコストです。

## エラー処理のマナー集

```go
// ❌ 握りつぶし(Python の except: pass に相当する大罪)
result, _ := doSomething()

// ❌ 二重報告(ログにも出して、さらに返す → 上流でもログに出て重複する)
if err != nil {
	log.Println(err)
	return err
}

// ✅ 処理するか、包んで返すか、どちらか一方だけ
if err != nil {
	return fmt.Errorf("集荷処理: %w", err)
}
```

## 🚇 完成コード: `express/day10/main.go`

```go
// Gopher Express — 事故報告書制度の導入
package main

import (
	"errors"
	"fmt"
)

var ErrNoRoute = errors.New("ルートが開通していません")

type OverloadError struct {
	Weight, Capacity float64
}

func (e *OverloadError) Error() string {
	return fmt.Sprintf("積載オーバー: %.1fkg(上限 %.1fkg)", e.Weight, e.Capacity)
}

var routes = map[string]float64{"north": 500, "south": 2000}

// 最下層: ルート確認と積載チェック
func loadCargo(dest string, weight float64) error {
	capacity, ok := routes[dest]
	if !ok {
		return ErrNoRoute
	}
	if weight > capacity {
		return &OverloadError{Weight: weight, Capacity: capacity}
	}
	return nil
}

// 中間層: 文脈を足してラップ
func ship(dest string, weight float64) error {
	if err := loadCargo(dest, weight); err != nil {
		return fmt.Errorf("%s 行き %.1fkg の出荷: %w", dest, weight, err)
	}
	fmt.Printf("✅ %s 行き %.1fkg 出荷完了\n", dest, weight)
	return nil
}

// 最上層: 種類ごとに対応を変える
func main() {
	orders := []struct {
		dest   string
		weight float64
	}{
		{"north", 320}, {"east", 10}, {"north", 800},
	}

	for _, o := range orders {
		err := ship(o.dest, o.weight)
		if err == nil {
			continue
		}
		var oe *OverloadError
		switch {
		case errors.Is(err, ErrNoRoute):
			fmt.Println("📋 対応: 開通予定表を確認 →", err)
		case errors.As(err, &oe):
			fmt.Printf("📋 対応: %.1fkg 減らして再出荷 → %v\n", oe.Weight-oe.Capacity, err)
		default:
			fmt.Println("📋 対応: 所長へエスカレーション →", err)
		}
	}
}
```

## 📝 今日の配達訓練(演習)

1. `ErrFragile`(割れ物は空輸不可)という番兵エラーを追加し、`errors.Is` で
   判別して専用メッセージを出してください。
2. `%w` を `%v` に変えると `errors.Is` が効かなくなることを確認してください
   (`%v` は文字列に混ぜ込むだけで、包んでいないからです)。
3. わざと `panic("トンネル崩落")` する配達関数を作り、`recover` で回収して
   後続の配達が続行されることを確認してください。前章の「nil の入った
   インターフェース」の罠が、エラー戻り値でこそ起きやすい理由も復習しましょう
   (カスタムエラー型のポインタを返すときが危険地帯です)。

---

台帳は `map[string]*Parcel` 専用、コンテナは `[]string` 専用……
「中身の型だけ違う同じ設備」を毎回書き直すのにうんざりしてきました。
Go が 10 年拒み続け、ついに導入した **ジェネリクス** の出番です。
→ [第11章 万能コンテナ](11_generics.md)
