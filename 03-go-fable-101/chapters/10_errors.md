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

## いつ error を返すべきか

すべての関数に `error` を付ける必要はありません。付けるべきは
**「失敗が現実に起こりうる操作」** だけです。

```go
// ✅ 失敗しうる → error を返す
func ParseWeight(s string) (float64, error) {
	return strconv.ParseFloat(s, 64) // 不正な文字列が来うる
}

// ❌ 起こりえない失敗のために error を足すのは過剰設計
func Double(x int) int {
	return x * 2 // 失敗しようがない
}
```

判断基準は明快です: **「呼び出し側がこの失敗を知って何か対処できるか」**。

| 状況 | 対応 |
|---|---|
| 外部要因(ファイル、ネットワーク、ユーザー入力)に依存し、失敗が起こりうる | `error` を返す |
| 純粋な計算で、前提さえ守られていれば絶対に成功する | `error` 不要 |
| 理論上は失敗しうるが、呼び出し側がその情報で何もできない | 設計を見直す価値あり(握りつぶされがち) |

これは後述する panic の基準(「呼び出し側に回復の手立てがあるか」)と表裏一体です。
回復の余地があるなら `error`、ないなら戻り値を増やさずに `panic` か、
そもそも失敗しない設計にします。

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

**ラップすべきかどうかの判断基準**は、「呼び出し元が下流のエラーの種別で
分岐する可能性があるか」です。

| 状況 | すべきこと |
|---|---|
| 呼び出し元が `errors.Is` / `errors.As` で種別判定する可能性がある | `%w` でラップ |
| ログ用の文言を作るだけで、呼び出し元は種別判定しない | `%w` でも `%v` でも実害はない |
| 下流のエラー型が実装の詳細で、外部に漏らしたくない(カプセル化したい) | あえてラップしない。自パッケージの語彙のエラーに変換して返す |

例えば公開 API の内部で使っている DB ドライバのエラー(`*pq.Error` など)をそのまま
`%w` で透過させると、呼び出し側がその内部実装に依存してしまいます。その場合は:

```go
func (r *Repo) Find(id string) (*Parcel, error) {
	p, err := r.db.QueryRow(...)
	if err != nil {
		return nil, ErrNotFound // 中身は隠し、自パッケージの語彙に変換する
	}
	return p, nil
}
```

のように **変換して隠す** 選択もあります。「ラップ = 常に正しい」ではなく、
「呼び出し側にどこまで内部情報を見せたいか」で決めます。

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

### panic が「明確に正解」な場面 / そうでない場面

`panic` は基本的に使いません。実務で妥当なのはだいたいこの3パターンに絞られます。

**① `MustXxx` 系のパッケージ初期化ヘルパー**

```go
var re = regexp.MustCompile(`^\d{3}-\d{4}$`) // 起動時に1回だけ実行
```

正規表現の文字列は自分で書いたコードの一部です。コンパイルに失敗するとしたら
それは実行時の異常事態ではなく **自分のコードのバグ**。`error` で運んでも
呼び出し側は何も対処できないので、即座にクラッシュさせて気づかせます。

**② 「ここに来たら絶対バグ」という不変条件のチェック**

```go
switch status {
case Pending, Shipped, Delivered:
	// ...
default:
	panic(fmt.Sprintf("未知のstatus: %v(型の追加漏れ)", status))
}
```

型(や定数)を追加したのに分岐を足し忘れた、というプログラマのミスを
実行時に検出する用途です。データの異常ではなく、コードの前提が崩れています。

**③ サーバーのリクエスト単位の防御(recover とセット)**

上の `safeDeliver` の例がまさにこれです。ただしこれは panic を **投げる** 用途
ではなく、コード側が想定していなかったバグ(nil デリファレンスや範囲外アクセス
など)で **勝手に発生した** panic を **受け止める** 用途である点に注意してください。
開発者が意図的に `panic()` と書くケースではありません。

**逆に panic を使うべきでない典型**

- ファイルが見つからない、ネットワークが切れた、ユーザー入力が不正
  → これらは日常的に起こりうる失敗なので、すべて `error`
- 起動時の致命的エラー(冒頭の `mustLoadConfig` 相当)も、実務では `panic` より
  `log.Fatal` の方が向くことが多いです。`panic` はスタックを巻き戻して defer を
  実行しますが、`log.Fatal` は内部で `os.Exit` を呼ぶだけで defer は一切
  実行されません。「defer によるクリーンアップをしてから終わりたいか」が
  使い分けの分かれ目です。

> まとめ: panic は「実行時に起こりうる失敗」ではなく「コード自体が壊れている」
> ことを表す道具、と覚えると判断がぶれません。

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
