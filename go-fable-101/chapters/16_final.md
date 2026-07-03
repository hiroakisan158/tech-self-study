# 第16章 卒業制作 — 配送管理システム完成

## 🚇 今日のお話

入社から 16 章。あなたは Gopher Express の全業務を Go 化してきました。
最終日の今日は、全部品を 1 つのシステムに組み上げ、**シングルバイナリとして出荷** します。

新しい文法はもう出てきません。この章のテーマは **「Go のプロジェクトを
どう構成し、どう仕上げ、どう配るか」** です。

## 完成品の仕様

**gx** — Gopher Express 配送管理システム

- `gx serve` — 電子受付(HTTP API)を起動
- `gx add --dest north --weight 3.2` — CLI から荷物を登録
- `gx list` — 台帳の一覧表示
- 台帳は JSON ファイルに永続化
- 配達シミュレーションは並行実行
- 全パッケージにテスト

## プロジェクト構成

```
express/
├── go.mod
├── cmd/
│   └── gx/
│       └── main.go        ← エントリポイント(薄く保つ)
├── internal/
│   ├── ledger/            ← 台帳(第5,8章)
│   │   ├── ledger.go
│   │   ├── store.go       ← JSON 永続化
│   │   └── ledger_test.go
│   ├── fleet/             ← 車両と配車(第9,12,13章)
│   │   ├── carrier.go
│   │   └── dispatch.go
│   └── api/               ← HTTP 受付(第14章)
│       ├── server.go
│       └── server_test.go
└── testdata/
```

2 つの慣習を新しく導入しています:

- **`cmd/<アプリ名>/main.go`**: main パッケージ置き場の定番。アプリが複数あっても
  `cmd/gx`, `cmd/gx-admin` と並べられます。main.go は「部品を組み立てて起動するだけ」の
  薄い接着剤に保ちます
- **`internal/`**: この名前のディレクトリ以下は、**このモジュールの外から import
  できない** ことを Go ツールチェーンが強制します。第6章の「小文字 = 非公開」の
  パッケージ版です。迷ったら internal に置き、公開ライブラリにしたくなったら
  外に出す——が安全側の運用です

> 🐍 **Python との違い①: `src/` レイアウトや `__init__.py` 群に相当する議論が短い**
> Go のプロジェクト構成の公式ガイダンスは「小さく始めよ。最初は main.go 1 枚でよい」です。
> `cmd/` と `internal/` さえ知っていれば、有名 OSS のリポジトリはだいたい読めます。

## 部品の組み立て — ハイライト

紙面の都合で全文ではなく、設計判断が詰まった箇所を抜粋します。
(残りはこれまでの章のコードの再配置です——ぜひ自分の手で組んでください)

### 永続化: インターフェースで差し替え可能に(第9章の実践)

```go
// internal/ledger/store.go
package ledger

// Store は台帳の保存先。JSON ファイルでもメモリでも DB でもよい。
type Store interface {
	Load() (map[string]*Parcel, error)
	Save(map[string]*Parcel) error
}

// JSONStore はファイルへ保存する本番用 Store。
type JSONStore struct{ Path string }

func (s *JSONStore) Load() (map[string]*Parcel, error) {
	data, err := os.ReadFile(s.Path)
	if errors.Is(err, os.ErrNotExist) {
		return make(map[string]*Parcel), nil // 初回起動: 空の台帳(ゼロ値の思想!)
	}
	if err != nil {
		return nil, fmt.Errorf("台帳 %s の読み込み: %w", s.Path, err)
	}
	var parcels map[string]*Parcel
	if err := json.Unmarshal(data, &parcels); err != nil {
		return nil, fmt.Errorf("台帳 %s の解析: %w", s.Path, err)
	}
	return parcels, nil
}

func (s *JSONStore) Save(parcels map[string]*Parcel) error {
	data, err := json.MarshalIndent(parcels, "", "  ")
	if err != nil {
		return err
	}
	return os.WriteFile(s.Path, data, 0o644)
}
```

テストでは `MemStore`(map を持つだけの偽物)を Store として渡します。
モックライブラリなしで差し替えが済むのは、Store を **使う側(ledger)が
小さく定義した** からです。

### 配達シミュレーション: 並行処理の集大成(第12,13章)

```go
// internal/fleet/dispatch.go
package fleet

// DeliverAll は未配達の荷物を配達員 workers 匹で並行配達する。
// ctx がキャンセルされたら安全に打ち切る。
func DeliverAll(ctx context.Context, parcels []*ledger.Parcel, workers int) int {
	jobs := make(chan *ledger.Parcel)
	var done atomic.Int64

	var wg sync.WaitGroup
	for i := 0; i < workers; i++ {
		wg.Add(1)
		go func() {
			defer wg.Done()
			for p := range jobs {
				time.Sleep(50 * time.Millisecond) // 配達時間(のつもり)
				p.MarkDelivered()
				done.Add(1)
			}
		}()
	}

	for _, p := range parcels {
		select {
		case jobs <- p:
		case <-ctx.Done():
			break
		}
	}
	close(jobs)
	wg.Wait()
	return int(done.Load())
}
```

### エントリポイント: flag によるサブコマンド

```go
// cmd/gx/main.go
package main

func main() {
	if len(os.Args) < 2 {
		fmt.Fprintln(os.Stderr, "usage: gx <serve|add|list>")
		os.Exit(2)
	}

	store := &ledger.JSONStore{Path: "ledger.json"}
	book, err := ledger.NewBookFrom(store)
	if err != nil {
		fmt.Fprintln(os.Stderr, "起動失敗:", err)
		os.Exit(1)
	}

	switch os.Args[1] {
	case "serve":
		err = api.Serve(":8080", book)
	case "add":
		fs := flag.NewFlagSet("add", flag.ExitOnError)
		dest := fs.String("dest", "", "行き先")
		weight := fs.Float64("weight", 0, "重量(kg)")
		fs.Parse(os.Args[2:])
		var p *ledger.Parcel
		if p, err = book.Add(*dest, *weight); err == nil {
			fmt.Println("受付:", p.ID)
		}
	case "list":
		for _, p := range book.All() {
			fmt.Println(p)
		}
	default:
		err = fmt.Errorf("不明なコマンド: %s", os.Args[1])
	}

	if err != nil {
		fmt.Fprintln(os.Stderr, "エラー:", err)
		os.Exit(1)
	}
}
```

標準の `flag` パッケージは argparse より簡素です(`--dest` と `-dest` は同じ扱い、
サブコマンドは手組み)。本格的な CLI には `spf13/cobra` が定番ですが、
この規模なら標準で十分——**まず標準ライブラリ、足りなくなったら外部** が Go の順序です。

## 出荷 — シングルバイナリの快感

```bash
go build -o gx ./cmd/gx
./gx add --dest north --weight 3.2
./gx list
./gx serve
```

`go build` が吐くのは **依存をすべて含んだ実行ファイル 1 個** です。
配布先に Go のインストールは不要。venv も、pip install も、
「相手の Python が 3.9 で動かない」もありません。

さらに:

```bash
GOOS=linux GOARCH=arm64 go build -o gx-pi ./cmd/gx     # Raspberry Pi 用
GOOS=windows GOARCH=amd64 go build -o gx.exe ./cmd/gx  # Windows 用
GOOS=darwin GOARCH=arm64 go build -o gx-mac ./cmd/gx   # Apple Silicon 用
```

環境変数 2 つで **クロスコンパイル** が完了します。Docker、kubectl、Terraform と
いったインフラツールがこぞって Go 製なのは、この配布の圧倒的な楽さが大きな理由です。

> 🔍 **なぜこんなにあっさりクロスコンパイルできるの?**
> Go は標準ライブラリまで含めて自前実装(C ライブラリ非依存)を貫いており、
> ランタイム(GC、goroutine スケジューラ)ごとバイナリに焼き込みます。
> バイナリが数 MB と大きめなのはその代償ですが、「動かす環境の事情に
> 依存しない」ことを優先した設計です。

## 卒業試験(最終演習)

1. **組み上げ**: この章の設計に従い、これまでの章の部品を移植して gx を完成させてください。
   `go test -race ./...` が全部通ることが合格ラインです。
2. **機能追加**: `gx deliver --workers 5` サブコマンドを追加し、`DeliverAll` を呼んで
   「N 件配達完了」を表示、台帳を保存してください。workers を変えて時間を計測しましょう。
3. **API 拡張**: `POST /deliver` を api パッケージに追加し、HTTP からも配達を
   起動できるようにしてください。同時に 2 回叩かれても台帳が壊れないことを
   `-race` で確認すること。
4. **クロスコンパイル**: 自分の OS 以外向けにビルドし、ファイルサイズと
   `file` コマンドの出力を観察してください。

## 🎓 卒業 — Python と Go、2 つの言語を持つあなたへ

16 章を通して見てきた Go の設計判断を 1 枚にまとめます。

| 場面 | Python の選択 | Go の選択 |
|---|---|---|
| 型 | 動的 + 任意の型ヒント | 静的、コンパイラが常時検査 |
| エラー | 例外で大域脱出 | 値として毎回その場で処理 |
| 継承 | 多重継承まである | 廃止。埋め込みとインターフェース |
| 抽象化 | ダックタイピング | 暗黙実装のインターフェース |
| 並行処理 | asyncio(協調的・単一コア) | goroutine(先取的・全コア) |
| 書き方の自由 | 複数の書き方を許す | 1 通りに寄せる(gofmt、for のみ) |
| 隠れた処理 | 便利さのため許容 | コストは見えるところに書かせる |
| 配布 | 環境ごと構築 | シングルバイナリ |

どちらが上ということではなく、**Python は「書く人の表現力」に、Go は
「読む人と運用する人の負担」に最適化した言語** です。探索的な分析や小さなツールは
Python で、複数人で長く運用するサーバーやインフラ CLI は Go で——2 つの言語を
使い分けられることが、あなたの武器になります。

### 次に読むもの

- [A Tour of Go](https://go.dev/tour/) — 公式チュートリアル(この教材の復習に)
- [Effective Go](https://go.dev/doc/effective_go) — 公式のイディオム集
- [Go Code Review Comments](https://go.dev/wiki/CodeReviewComments) — レビューで指摘される事項の先回り
- [100 Go Mistakes and How to Avoid Them](https://100go.co/) — 本教材の落とし穴コーナーの完全版
- 標準ライブラリのソースコード — Go はソースが読みやすい言語です。`io` と `errors` から

Gopher Express は今日も地下トンネルで荷物を運んでいます。
あなたの書いた並行配達システムに乗って。🚇🐹

**卒業おめでとうございます!**
