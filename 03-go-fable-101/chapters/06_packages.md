# 第6章 営業所を増やす — パッケージと go mod

## 🚇 今日のお話

Gopher Express は 2 号店を出せるほど成長しました。1 つの部屋(ファイル)に
台帳係も料金係も詰め込むのは限界です。業務を **パッケージ** という部屋に分け、
会社全体を **モジュール** として登記します。

## モジュールの登記 — go mod

Go のプロジェクトは **モジュール** という単位で管理します。ルートで一度だけ:

```bash
mkdir express && cd express
go mod init gopher.example/express
```

これで `go.mod` ができます。中身はたった数行です。

```
module gopher.example/express

go 1.22
```

- モジュール名は慣習的に「リポジトリの URL」(例: `github.com/you/express`)にします。
  公開予定がなければ何でも構いません
- 外部ライブラリは `go get github.com/...` で追加され、`go.mod` と、
  検証用ハッシュを記録する `go.sum` に自動記録されます

> 🐍 **Python との違い①: venv も requirements.txt も要らない**
> Python では venv を作り、pip で入れ、requirements.txt(や pyproject.toml)を
> 手で保守しました。Go は `go.mod` 1 つで完結し、**仮想環境という概念自体がありません**。
> 依存はバージョンごとにマシン共通のキャッシュに保存され、ビルド時に
> モジュールごとの `go.mod` が正確なバージョンを指名します。
> 環境の分離を「ディレクトリのコピー」でなく「宣言ファイル」で実現している形です。
>
> 🔍 **なぜ最初からそうじゃなかったの?** 実は Go にも暗黒期があり、
> 2018 年(Go 1.11)まで「すべてのコードを GOPATH という 1 つの場所に置き、
> バージョン指定は不可能」という仕組みでした。ネットの古い記事で GOPATH の
> 設定を延々説明しているものは、この時代の遺物です。現代の Go では
> **GOPATH の設定は不要** です。

## パッケージ = ディレクトリ

Go では **1 ディレクトリ = 1 パッケージ** です。ディレクトリ内の全 `.go` ファイルは
同じ `package` 宣言を持ち、**ファイル同士は import なしで互いの名前が見えます**。

```
express/
├── go.mod
├── main.go            ← package main(起動役)
├── ledger/
│   └── ledger.go      ← package ledger(台帳係)
└── fare/
    ├── fare.go        ← package fare(料金係)
    └── discount.go    ← package fare(同じ部屋の別ファイル)
```

```go
// fare/fare.go
package fare

// Calc は重量から運賃を計算する。
func Calc(weight float64) int {
	return baseFare + int(weight*20)
}

const baseFare = 500 // 小文字 → このパッケージの外からは見えない
```

```go
// main.go
package main

import (
	"fmt"

	"gopher.example/express/fare" // モジュール名 + ディレクトリで import
)

func main() {
	fmt.Println(fare.Calc(3.2)) // パッケージ名.名前 で呼ぶ
}
```

> 🐍 **Python との違い②: ファイルはモジュールではない**
> Python では 1 ファイル = 1 モジュールで、`from fare.discount import apply` のように
> ファイル単位で import しました。Go の import 単位は **ディレクトリ** で、
> 中のファイル分割は完全に内部事情です(外からはどのファイルに書いたか見えません)。
> また `from x import *` に相当するものはなく、使う側は常に `fare.Calc` と
> パッケージ名を付けて呼びます。名前がどこから来たか、常にコードに書いてある状態です。

## 大文字 = 公開、小文字 = 非公開

Go には `public` / `private` キーワードがありません。
**識別子の頭文字が大文字なら公開(exported)、小文字なら非公開** です。

```go
func Calc(...)     // 外から使える
func roundUp(...)  // このパッケージ専用
type Parcel struct {
	ID     string  // 外から読み書きできる
	secret string  // パッケージ外からは見えない
}
```

> 🔍 **なぜそうなっているの? — 命名を仕様にした理由**
> Python の `_private` は「触るな」の紳士協定で、実際には触れました。
> Java の `public` は宣言を見に行かないと分かりません。Go は
> **呼び出し側のコードを見ただけで公開・非公開が分かる** ことを選びました。
> `fare.Calc` は必ず公開 API、`roundUp` と小文字で呼べているなら必ず同一パッケージ内。
> キーワードを増やさずに情報を名前自体に埋め込む、Go らしい割り切りです。
> 副作用として「頭文字を変えると公開範囲が変わる」ため、リネームが API 変更になります。

## 循環 import は禁止

パッケージ A が B を import し、B が A を import する——Python では(気をつければ)
動くこともありますが、Go は **コンパイルエラー** です。

> 🔍 **なぜ禁止なの?** 循環がなければ、パッケージは依存順に 1 回ずつ
> 並列コンパイルできます。Go が巨大コードベースでもビルドが速いのは、
> この制約のおかげでもあります(Go 誕生の動機の 1 つが、Google での
> C++ の絶望的なビルド時間でした)。循環が欲しくなったら設計の
> 見直しどき——共通部分を第 3 のパッケージに切り出すのが定石です。

## init と import の副作用

パッケージには `func init()` という特殊関数を置けます。import されたとき、
`main` より先に自動実行されます(Python のモジュールトップレベルコードに相当)。
乱用すると初期化順が追えなくなるため、現代の Go では最小限にするのが良い習慣です。

## 道具箱 — 毎日使うコマンド

| コマンド | 役割 | Python でいうと |
|---|---|---|
| `go run .` | その場で実行 | `python main.py` |
| `go build` | 実行ファイル生成 | (相当なし) |
| `go fmt ./...` | コード整形 | `black` |
| `go vet ./...` | 静的検査 | `flake8` / `mypy` の一部 |
| `go test ./...` | テスト実行 | `pytest` |
| `go get pkg@v1.2.3` | 依存追加 | `pip install` |
| `go mod tidy` | 依存の整理 | `pip freeze` の逆方向 |

`./...` は「カレント以下の全パッケージ」の意味です。
`gofmt`(と import 整理を加えた `goimports`)は **設定項目が存在しない** のが特徴で、
Go コミュニティにインデント論争はありません。エディタの保存時整形を必ず有効にしましょう。

## 🚇 完成コード: 2 号店開設

```go
// ledger/ledger.go
package ledger

import "fmt"

// Parcel は 1 つの荷物のカルテ。
type Parcel struct {
	ID      string
	Dest    string
	Weight  float64
	Fragile bool
}

// Book は伝票番号で荷物を管理する台帳。
type Book struct {
	parcels map[string]Parcel // 非公開: 直接触らせない
	nextID  int
}

// NewBook は空の台帳を作る(Go 流コンストラクタ。慣習的に New〜と名付ける)。
func NewBook() *Book {
	return &Book{parcels: make(map[string]Parcel)}
}

// Add は荷物を登録し、発行した伝票番号を返す。
func (b *Book) Add(dest string, weight float64, fragile bool) string {
	b.nextID++
	id := fmt.Sprintf("GX-%04d", b.nextID)
	b.parcels[id] = Parcel{ID: id, Dest: dest, Weight: weight, Fragile: fragile}
	return id
}

// Find は伝票番号で荷物を探す。
func (b *Book) Find(id string) (Parcel, bool) {
	p, ok := b.parcels[id]
	return p, ok
}
```

`(b *Book)` という見慣れない書き方(メソッドとポインタレシーバ)が出てきましたが、
これが次の 2 章のテーマです。「NewBook がなぜ `*Book` を返すのか」も含めて、
今は写経で OK です。

```go
// main.go
package main

import (
	"fmt"

	"gopher.example/express/ledger"
)

func main() {
	book := ledger.NewBook()
	id := book.Add("north", 3.2, true)
	fmt.Println("受付完了:", id)

	if p, ok := book.Find(id); ok {
		fmt.Printf("照会: %+v\n", p)
	}
}
```

```bash
go run .
```

## 📝 今日の配達訓練(演習)

1. `fare` パッケージを作り、第3章の `findFare` を移植して `main` から呼んでください。
   小文字のままだと import 先から見えないことをエラーで確認しましょう。
2. `go fmt ./...` をわざと崩したコード(インデント滅茶苦茶)に掛けて、
   一発で直ることを確認してください。
3. `ledger` から `fare` を import し、さらに `fare` から `ledger` を import して、
   循環 import のエラーメッセージを観察してください(確認したら片方は消しましょう)。

---

**🌿 ここから中級編です。** 台帳の `NewBook` はなぜ `*Book` を返すのか?
`(b *Book)` の `*` は何なのか? Python にはなかった概念、**ポインタ** と
正面から向き合います。怖くありません——Python の「参照」を表に出しただけです。
→ [第7章 現物とコピー](07_pointers.md)
