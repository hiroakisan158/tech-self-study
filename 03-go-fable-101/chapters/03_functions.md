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

> ⚠️ **注意: `Close()` を呼び忘れるとどうなる?**
> `defer` はあくまで「`Close()` を呼び忘れないための予約」であって、閉じているのは
> `Close()` の呼び出しそのものです。もし `defer f.Close()` すら書かず放置すると、
> fd(ファイルディスクリプタ)やコネクションは **プロセス終了までOSレベルで
> 解放されません**。`os.File` には GC 時に閉じる `runtime.SetFinalizer` の保険が
> 一応ありますが、GC が走るタイミングはメモリ確保の圧力次第で不定です。
> 長時間動くサーバーだと、fd を開けっぱなしにし続けて `too many open files` に
> 直結する事故になり得ます。コンパイラも `Close()` の呼び忘れを教えてくれないので、
> **リソースを取得したらその場ですぐ `defer Close()` を書く** のがGoの鉄則です。

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

いきなり難しく見える機能ですが、**3段階**に分けると簡単です。
「① 関数も値である」→「② 関数を作って返す関数がある」→「③ その関数が変数を覚えている(クロージャ)」
の順に見ていきましょう。

### ① まず「関数は値」を体感する

`int` や `string` の値を変数に入れられるように、Go では **関数そのものも値として変数に入れられます**。

```go
func shout(msg string) {
	fmt.Println(msg + "!!")
}

func main() {
	// shout という関数を、そのまま greet という変数に代入する
	greet := shout
	greet("こんにちは") // shout("こんにちは") と全く同じ意味 → こんにちは!!
}
```

`greet` には「文字列」でも「数値」でもなく「`shout`という処理そのもの」が入っています。
これが「関数は値」の意味です。値なので、変数に入れられるし、他の関数の引数としても渡せます。

### ② 「関数を返す関数」を作る

値であるなら、`return` で関数を返すこともできます。試しに、
**「挨拶の相手を決めて、挨拶する関数を返す」** 関数を作ってみます。

```go
// dest 行きの荷物用の挨拶メッセージを作る「関数」を返す関数
func makeGreeter(dest string) func() {
	return func() {
		fmt.Println(dest + " 行きの荷物です")
	}
}

func main() {
	northGreeter := makeGreeter("north") // 「north用の挨拶をする関数」を受け取る
	northGreeter()                       // north 行きの荷物です
}
```

実はこの時点ですでに`dest`は返された無名関数に**捕まえられています**。`makeGreeter`の実行は
`return`した瞬間に終わっているのに、後で`northGreeter()`を呼んだときにまだ`"north"`という値を
覚えていられるのは、`dest`が捕まえられているからです。これも立派なクロージャです。

> ⚠️ **クロージャになるかどうかの判定ルールは1つだけ**
> 「引数を渡したら自動で記憶される」わけではありません。ルールはこれだけです:
>
> > ある関数の中に**別の関数リテラル(`func(){...}`)**を書き、その中身が**外側のスコープの変数**を
> > 使っていたら、その内側の関数はクロージャになる。
>
> 外側の変数が「引数」か「ローカル変数(`:=`で作った変数)」かは関係ありません。
> 比べてみましょう:
>
> ```go
> // クロージャじゃない: 内側に無名関数が無い、ただの普通の関数
> func greet(name string) {
> 	fmt.Println("hello " + name)
> }
>
> // クロージャ: 内側の無名関数が外側の name を参照している
> func makeGreeter(name string) func() {
> 	return func() { // ← 内側の無名関数
> 		fmt.Println("hello " + name) // ← name を参照 → 捕まる
> 	}
> }
> ```
>
> `greet`が引数`name`を受け取っていること自体はクロージャと無関係です。分かれ目は
> 「内側に無名関数があり、それが外側の変数を使っているかどうか」だけ。Go自身がコードを見て
> 「このコードは外の変数を参照しているな」と自動判定してくれるので、C++のラムダの`[&n]`のような
> 「これを捕まえます」という明示的な宣言(キャプチャリスト)は不要です。**無名関数の中で、
> 外の変数名をそのまま使うだけ** — それがクロージャの構文の全てです。

`makeGreeter`の`dest`は**読むだけ**で書き換えていないので、地味に見えたかもしれません。
次のステップでは、捕まえた変数を**書き換えて、複数回の呼び出しにまたがって状態を保持する**、
クロージャらしい使い方を見せます。

### ③ クロージャ = 外側の変数を「覚えて・書き換えられる」関数

いよいよ本題です。以下の`newCounter`は、**呼ぶたびに 1, 2, 3... と増えていく番号を返す関数**を作ります。

```go
// 連番の伝票番号を発行するカウンタを作る
func newCounter() func() int {
	n := 0            // ← この n を、下の無名関数が「覚えて」しまう
	return func() int { // n を捕まえた関数(=クロージャ)
		n++
		return n
	}
}

next := newCounter()
fmt.Println(next()) // 1
fmt.Println(next()) // 2
fmt.Println(next()) // 3
```

**何が起きているか、1行ずつ追ってみましょう:**

1. `newCounter()` が呼ばれると、その場に `n := 0` という変数が作られる。
2. `return func() int { ... }` で、**`n` を内部で使っている無名関数**を作って返す。
   このとき、無名関数は `n` という変数を(値としてコピーするのではなく)**そのまま参照し続ける**。
3. `newCounter()` の実行は終わっているのに、`n` は消えずに生き残る。
   なぜなら返された無名関数がまだ `n` を使っているから(Go のランタイムが面倒を見てくれます)。
4. `next := newCounter()` で、その「`n`を覚えている関数」を `next` という変数に入れる。
5. `next()` を呼ぶたびに、`n++` が実行され、**同じ `n` が使い回される**。だから 1, 2, 3 と増える。

> ⚠️ **初見だとトリッキーなポイント: `n := 0` は2回目以降の`next()`では通らない**
> 「`next()`を呼ぶたびに`n := 0`も毎回実行されて、`n`がリセットされるのでは?」と誤解しやすい
> ところです。実際はそうなりません。理由は、**`n := 0`と`return func() int {...}`が
> 「別の関数」に属している**からです。
>
> ```go
> func newCounter() func() int {
> 	n := 0            // ← newCounter 自身の中身。newCounter() が呼ばれた瞬間に1回だけ実行される
> 	return func() int { // ← ここから下は「別の関数(クロージャ)」の中身
> 		n++              // ← next() を呼ぶたびに実行されるのはこの中だけ
> 		return n
> 	}
> }
> ```
>
> `next := newCounter()`で`next`に入るのは**内側の関数だけ**であり、`n := 0`を含む
> `newCounter`本体はもう含まれていません。だから`next()`を何回呼んでも、実行されるのは
> `n++`と`return n`だけで、`n := 0`は二度と通りません。実行順序を書き出すとこうなります:
>
> ```
> newCounter() が呼ばれる
>   → n := 0 が実行される(n が作られる、これが最初で最後)
>   → func() int {...} が作られて返される(まだ中身は実行されない)
> next := newCounter() で、その関数が next に入る
>
> next() を呼ぶ → n++ (n: 0→1), return 1   ( n := 0 は通らない)
> next() を呼ぶ → n++ (n: 1→2), return 2   ( n := 0 は通らない)
> next() を呼ぶ → n++ (n: 2→3), return 3   ( n := 0 は通らない)
> ```
>
> もし`n := 0`が`next()`のたびに実行されていたら、`n`は毎回0にリセットされて常に`1`しか
> 返ってこないはずです。実際には`1, 2, 3`と増え続けること自体が、`n := 0`が
> **最初の`newCounter()`呼び出し時にしか実行されない**ことの証拠です。

もし `newCounter()` をもう一度呼べば、**全く別の新しい `n`** が作られるので、
2つのカウンターは互いに影響しません。

```go
a := newCounter()
b := newCounter()
fmt.Println(a(), a(), b()) // 1 2 1  ← a と b は別々の n を持っている
```

このように、**「関数の外(正確には自分を作った関数のスコープ)にある変数を、
自分の中に閉じ込めて(closure = 閉包)覚え続ける関数」** をクロージャと呼びます。
`n` という「状態」を持ちながら動く関数、というイメージです。

> 🔍 **中では何が起きている? 「関数=ポインタ」で理解する**
> 変数に関数を入れると言っても、実体はコードそのものが変数にコピーされるわけではありません。
>
> - **クロージャじゃない普通の関数**(①の`greet := shout`)の場合、`greet`の中身は
>   実質的に **`shout`のコードの場所を指すポインタ** です。C言語の関数ポインタと同じイメージです。
> - **クロージャ**(`newCounter`が返す関数)の場合は、コードへのポインタだけでは
>   `n`がどこにあるか分からず情報不足です。そこでGoランタイムは裏側で
>   `[ コードへのポインタ, n へのポインタ ]` という小さな構造体をヒープ上に作り、
>   `next`にはその**構造体を指すポインタ**が入ります。
>
> これで③の1〜5の流れが繋がります。`newCounter()`を抜けても`n`が生き残っていたのは、
> `n`がスタックではなく**ヒープに確保**され、クロージャがそこへのポインタを握り続けているから
> です(コンパイラがこれを自動判定する仕組みは「エスケープ解析」と呼ばれます)。
> `a`と`b`が別々に数えられたのも、それぞれが**別々の`n`を指すポインタ**を持つ、
> 別々のクロージャ構造体だからです。
>
> | 変数の中身 | 普通の関数値 | クロージャ |
> |---|---|---|
> | 実体 | コードへのポインタ | コードへのポインタ + 捕まえた変数へのポインタ |
> | 例え | C言語の関数ポインタ | 関数ポインタ + それが参照する「持ち物袋」 |

> 🐍 Python でも同じことができますが、クロージャの中で外側の変数を**書き換える**には
> `nonlocal n` という宣言が必須でした(書かないと新しい変数だと誤解される)。
> Go では変数宣言(`:=`)と代入(`=`)が構文としてはっきり別れているので、
> `n++` と書くだけで「これは書き換えだ」と一意に伝わり、`nonlocal` 相当のものは不要です。
> なお Go には `lambda` に相当する省略記法はなく、無名関数も `func(x int) int { ... }` と
> フルで書きます。デコレータ構文もありませんが、「関数を受け取って関数を返す」こと自体は
> 普通にできます(演習 3)。

### 実務ではどう使う?

「struct を定義してメソッドを生やす」ほど大げさにせずに、**状態を持った関数**を
手軽に作れるのがクロージャの実務的なうまみです。よくある4パターンを紹介します。

**1. ハンドラファクトリ(依存を都度引数で渡さずに済む)**

Go の標準 Web サーバーは、URL ごとに処理関数(ハンドラ)を登録しますが、
その関数の**シグネチャ(型)は固定**で `func(w, r)` の2引数しか受け付けません。
なので `func(w, r, db *sql.DB)` のように**引数を増やして DB 接続を渡す、ということができません**。
`http.HandleFunc`側が`func(w, r)`の形しか受け付けないからです。

**ダメな例: グローバル変数で誤魔化す**

引数で渡せないなら、と安易にパッケージ変数に逃がすと動きはしますが、
「このハンドラがどの DB に依存しているか」がコードを読むだけでは分からなくなり、
テスト時に別の DB へ差し替えることも難しくなります。

```go
var globalDB *sql.DB // 😨 どこからでも書き換えられる、依存が見えない

func userHandler(w http.ResponseWriter, r *http.Request) {
	var name string
	globalDB.QueryRow("SELECT name FROM users WHERE id = ?", r.PathValue("id")).Scan(&name)
	// ...
}
```

**クロージャで解決する**

「引数として渡せないなら、関数を作る"前"に渡してしまえばいい」というのがクロージャの発想です。

```go
package main

import (
	"database/sql"
	"fmt"
	"log"
	"net/http"

	_ "github.com/mattn/go-sqlite3" // ドライバを sql パッケージに登録するためだけの import(直接は使わない)
)

// makeUserHandler は db を受け取り、それを「捕まえた」ハンドラ関数を返す工場。
// この関数自体が呼ばれるのは main() での登録時に1回だけ。
func makeUserHandler(db *sql.DB) http.HandlerFunc {
	// ここから内側が実際に返される無名関数(=クロージャ)。
	// db はこの関数定義の外(makeUserHandler の引数)にあるが、参照しているので捕まえられる。
	return func(w http.ResponseWriter, r *http.Request) { // ← http.HandlerFunc の形はちゃんと守る
		var name string // Scan の書き込み先。中身は QueryRow が成功したときだけ埋まる

		// QueryRow: SQLを実行して1行だけ取得する。
		// r.PathValue("id") は URL の {id} 部分(例: /users/42 なら "42")。
		// Scan(&name): 取得した行の値をポインタ経由で name に書き込む。
		err := db.QueryRow("SELECT name FROM users WHERE id = ?", r.PathValue("id")).Scan(&name)

		switch {
		case err == sql.ErrNoRows:
			// 該当する id の行が無かった場合の専用エラー値。DB障害とは区別して 404 を返す
			http.Error(w, "user not found", http.StatusNotFound)
			return
		case err != nil:
			// 接続断など、想定外のエラー。詳細はログに残し、クライアントには 500 だけ返す
			http.Error(w, "internal error", http.StatusInternalServerError)
			return
		}

		// ここまで来ていれば name には値が入っている
		fmt.Fprintf(w, "user: %s\n", name) // w に書き込んだ内容がそのままレスポンスボディになる
	}
}

func main() {
	// ★ DB接続(正確にはコネクションプール)はここで1回だけ確立する。
	// リクエストのたびにここを通るわけではない — main() は起動時に1度しか実行されない
	db, err := sql.Open("sqlite3", "app.db")
	if err != nil {
		log.Fatal(err) // 接続に失敗したら起動自体を諦める
	}
	defer db.Close() // main() を抜けるとき(=プロセス終了時)に閉じる予約。開けたら閉じるが鉄則

	// makeUserHandler(db) はここで1回だけ呼ばれ、db を捕まえたクロージャが返る。
	// 以降のリクエストで実際に呼ばれるのは、その返ってきたクロージャの方
	http.HandleFunc("/users/{id}", makeUserHandler(db))

	log.Println("listening on :8080")
	log.Fatal(http.ListenAndServe(":8080", nil)) // ここでリクエスト受付ループに入り、ブロックする
}
```

補足:

- **`db.QueryRow(...).Scan(&name)`** — SQLを実行して1行取得し、その値をポインタ経由で`name`に書き込む。
- **`sql.ErrNoRows`** — 該当行が無かった場合の専用エラー値。見つからない場合とDB障害を分けて処理できる。
- **`_ "github.com/mattn/go-sqlite3"`** — ドライバ自体は直接呼び出さず、読み込むだけで`init()`が
  `sql`パッケージに自分を登録する。使わないのにimportする印として`_`(ブランク識別子)を付ける。

> 💡 **同じ`db`を使い回すことの本当のメリットは「メモリ節約」ではない**
> 「リクエストのたびに接続済みの`db`が再利用されるから、都度接続しなくて済んでメモリが節約できる」
> と思うかもしれませんが、節約されているのは主に**メモリではなく「時間」と「DBの接続数上限」**です。
>
> - DB接続の確立は TCP ハンドシェイク + 認証を伴う**わりと重い処理**です。もしハンドラの中で
>   毎回 `sql.Open(...)` していたら、リクエストごとに数ミリ秒〜数十ミリ秒の遅延が余計にかかります。
> - DB サーバー側には**同時接続数の上限**があります(例: PostgreSQL のデフォルトは
>   `max_connections` ≈ 100)。リクエストのたびに新しい接続を張っていたら、アクセスが増えた瞬間に
>   この上限に達してエラーになります。
> - そもそも `*sql.DB` という型自体、単一の接続ではなく**コネクションプール**(複数の実接続を
>   まとめて管理する仕組み)です。`QueryRow`のたびにプールから空いている接続を1本借りて、
>   使い終わったら返す、という動きを内部でやってくれています。接続の使い回し自体は
>   標準ライブラリがすでに面倒を見てくれる部分で、クロージャの役目は
>   「その**プール管理オブジェクト自体を1個だけ作って、みんなで共有する**」ことです。
>
> もし`sql.Open`をハンドラの中(リクエストのたび)に書いてしまったら、リクエストごとに
> 新しいプールを作ることになり、遅延が増えるだけでなく、古い接続がちゃんと閉じられずに
> 残り続けて接続数の上限に張り付く、という実務上よくある事故(コネクションリーク)につながります。

> 🔍 **ずっと使い回していて expire しない?**
> 個々の物理接続はexpireし得ます。DBサーバー側のアイドルタイムアウト(MySQLの`wait_timeout`など)や、
> 間に挟まるロードバランサー・NATゲートウェイ・プロキシ(PgBouncer、AWS RDS Proxyなど)が
> 一定時間使われていない接続を勝手に切断することがあるためです。
>
> Goの`database/sql`はこれをある程度自動で吸収します。プールから借りた接続が実は既に
> 死んでいた場合(`driver.ErrBadConn`)、クエリの送信がまだ始まっていない段階なら
> **自動的に別の接続で1回だけリトライ**してくれます。ただしこれは万能ではないため、
> 実務では接続を能動的に入れ替える設定をしておくのがベストプラクティスです。
>
> ```go
> db.SetConnMaxLifetime(5 * time.Minute) // 生存5分を超えた接続は次に使うとき強制的に張り直す
> db.SetConnMaxIdleTime(1 * time.Minute) // 1分アイドルなら接続を閉じる
> db.SetMaxOpenConns(25)                 // 同時に開ける接続数の上限
> db.SetMaxIdleConns(25)                 // プールに保持しておくアイドル接続数
> ```
>
> `SetConnMaxLifetime`をロードバランサーやDBサーバー側のタイムアウトより短く設定しておけば、
> 「向こうが勝手に切る前にこちらから張り替える」形になり、`invalid connection`のような
> エラーを予防できます。つまり「1回作ったらずっとそのまま」ではなく、`*sql.DB`は内部で
> 複数の接続を管理し、古くなったものは自動または設定に従って入れ替えている、というのが正確です。
> クロージャが捕まえているのは「その入れ替えも含めて面倒を見てくれるプール管理オブジェクトそのもの」
> なので、`db`自体を使い回すことと個々の物理接続がexpireすることは別レイヤーの話です。

**何が起きているか:**

1. `makeUserHandler(db)` を呼ぶと、`db` は返される無名関数に**捕まえられる**
   (③の`newCounter`と全く同じ仕組み)。
2. 返されるのは `func(w, r)` という**正しい形の関数**なので `http.HandleFunc` に渡せる。
3. リクエストが来るたびに Go のサーバーがこの関数を呼び出し、中では捕まえておいた
   **同じ`db`**を毎回使い回せる。

> ⚠️ **勘違いしやすい点: `makeUserHandler(db)` は1回しか呼ばれない**
> 2回目以降のリクエストで呼ばれるのは `makeUserHandler` 自体ではなく、
> **そこで返された中身の無名関数(クロージャ)** です。イメージとしては、
> `makeUserHandler` は「工場」で登録時に1回だけ動き、工場が作った「製品」(クロージャ)の方が
> リクエストの数だけ繰り返し動きます。`net/http`が内部でこの繰り返し呼び出しを
> 自動でやってくれるので、自分でループを書く必要はありません。

グローバル変数版と結果はほぼ同じに見えますが、決定的な違いは
**「このハンドラは`db`に依存している」という事実が関数のシグネチャ(`makeUserHandler(db *sql.DB)`)
に明記される**ことです。テストのときは本物の代わりにテスト用のDBを渡すだけで済み、
グローバル変数のように「どこかで書き換えられていないか」を心配する必要もありません。

複数の依存(DBとロガーなど)がある場合も、同じ考え方で拡張できます。

```go
func makeUserHandler(db *sql.DB, logger *log.Logger) http.HandlerFunc {
	return func(w http.ResponseWriter, r *http.Request) {
		logger.Println("user requested:", r.PathValue("id"))
		var name string
		db.QueryRow("SELECT name FROM users WHERE id = ?", r.PathValue("id")).Scan(&name)
		// ...
	}
}
```

**2. ミドルウェア(演習 3 の「デコレータもどき」がまさにこれ)**

ログ出力・認証チェックなどを「関数を関数でラップする」形で差し込みます。

```go
// withLogging は「次に呼ぶべきハンドラ」を受け取り、前後にログ処理を足した
// 新しいハンドラを返す。next 自体も http.HandlerFunc という「関数」なので、
// 関数を引数に取り、関数を返すこの形が作れる(①〜③で見た「関数は値」の応用)
func withLogging(next http.HandlerFunc) http.HandlerFunc {
	// この無名関数が捕まえているのは next だけ(外側 = withLogging の引数だから)。
	// next は「本来実行したかった処理」そのものが入っている変数
	return func(w http.ResponseWriter, r *http.Request) {
		// start はこの無名関数の "内側" で宣言されているので捕まえられていない。
		// リクエストが来るたびに、まっさらな start がゼロから作られる
		start := time.Now() // リクエスト受信時刻を記録(処理時間計測の起点)

		next(w, r) // 捕まえておいた本来のハンドラ(例: makeUserHandler が返したもの)を実行

		// next の処理が完了した後にここに戻ってくる。処理前後を挟めるのがラップの利点
		log.Println(r.URL.Path, time.Since(start)) // かかった時間をログに出力
	}
}

// withLogging(makeUserHandler(db)) は内側から評価される:
//   1. makeUserHandler(db)  → db を捕まえたハンドラ(クロージャ)ができる
//   2. withLogging(それ)    → ①をさらに捕まえた、ログ付きハンドラ(クロージャのクロージャ)ができる
// http.HandleFunc に渡す時点では、db 接続もログ機能も1つの関数値に折りたたまれている
http.HandleFunc("/users", withLogging(makeUserHandler(db)))
```

> 🔍 **`next`って結局なに? — 「dbを持ち運んでいる」ように見えるカラクリ**
> `withLogging`は`db`について**何も知りません**。`next`が捕まえているのは「dbの情報」では
> なく、**「dbをすでに内部に持っている、1個の関数」**です。`withLogging(makeUserHandler(db))`は
> 内側から評価されるので、実際に何が起きているかを層で分けて追いましょう。
>
> ```
> ① makeUserHandler(db) が先に呼ばれる
>    → db を捕まえた関数ができる。これを handlerA と呼ぶ
>    → handlerA の中身: func(w, r) { db.QueryRow(...) ... }  ← db は既に "焼き込まれて" いる
>
> ② withLogging(handlerA) が呼ばれる
>    → withLogging の引数 next には handlerA が入る(= next は db を持つ関数を指している)
>    → withLogging は新しい関数を返す。これを handlerB と呼ぶ
>    → handlerB の中身: func(w, r) { start := ...; next(w, r); log...() }
> ```
>
> `withLogging`から見れば`next`はただの「`func(w, r)`という形をした、何かをする関数」でしか
> なく、中身が`db`を使っていようがいまいが関係なく同じようにラップできます。中身を知らなくて
> いい、というのがこのパターンの本質です。
>
> リクエストが来たときの実際の流れはこうなります:
>
> ```
> 1. net/http が handlerB(w, r) を呼ぶ
> 2. handlerB の中で start が記録される
> 3. handlerB の中で next(w, r) が呼ばれる → これは実質 handlerA(w, r)
> 4. handlerA(w, r) の中で、handlerA 自身が捕まえていた db が使われる ← ここで初めて db が登場
> 5. handlerA の処理が終わって handlerB に戻り、ログが出力される
> ```
>
> つまり「`db`接続情報を保持しつつラップできる」ように**見える**のは、`next`(=`handlerA`)
> 自体が「クロージャ = コードへのポインタ + 捕まえた変数へのポインタ」という構造を持つ**値**
> だからです(この構造は先の🔍「関数=ポインタ」で説明した通り)。`withLogging`はその**値を
> まるごと**受け取って捕まえているだけで、`db`を直接扱っているわけではありません。
>
> なお`next`という名前自体は予約語ではなく、**「次に呼ぶハンドラ」を表す慣習的な命名**に
> 過ぎません。関数型の引数名は文脈に応じて自由に付けられます(比較関数なら`less`、
> 汎用コールバックなら`f`、イベントハンドラなら`onClick`など)。

**3. ソートの比較条件を持ち運ぶ**

`sort.Slice` は比較関数を受け取りますが、外側のデータを捕まえたクロージャとして書くのが定番です。

```go
sort.Slice(orders, func(i, j int) bool {
	return orders[i].weight < orders[j].weight // orders を捕まえている
})
```

**4. カウンタ・キャッシュなど「状態を持つ関数」を手軽に作る**

まさに `newCounter`/`newTicketIssuer` がこれです。

```go
func memoize(f func(int) int) func(int) int {
	cache := map[int]int{}
	return func(n int) int {
		if v, ok := cache[n]; ok {
			return v
		}
		v := f(n)
		cache[n] = v
		return v
	}
}
```

共通しているのは、「外部から都度渡すには面倒な状態(DB接続、次に呼ぶ関数、キャッシュ)を、
関数自身に持たせて隠蔽できる」という点です。

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
