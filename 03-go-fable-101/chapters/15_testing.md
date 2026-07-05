# 第15章 出荷前検査 — go test とテーブル駆動テスト

## 🚇 今日のお話

Gopher Express は品質保証部を設立します。運賃計算が 1 ゴールドずれれば苦情、
台帳が壊れれば営業停止。**すべての設備に出荷前検査(テスト)** を義務付けます。

Go のテストは言語とツールチェーンに最初から組み込まれています。
pytest のような外部ツールは要りません——そして書き味もかなり違います。

## テストの基本ルール

1. テストは `_test.go` で終わるファイルに書く(同じパッケージ内に置く)
2. テスト関数は `func TestXxx(t *testing.T)` という形
3. `go test ./...` で全部走る

```go
// fare/fare.go
package fare

func Calc(weight float64) int {
	return 500 + int(weight*20)
}
```

```go
// fare/fare_test.go
package fare

import "testing"

func TestCalc(t *testing.T) {
	got := Calc(3.2)
	want := 564
	if got != want {
		t.Errorf("Calc(3.2) = %d; want %d", got, want)
	}
}
```

```bash
go test ./...        # 全パッケージのテスト
go test -v ./fare    # 詳細表示
go test -run Calc    # 名前で絞り込み
```

> 🐍 **Python との違い①: assert がない**
> pytest の `assert got == want` に相当する構文は Go の標準テストにありません。
> **普通の if で比較して、ダメなら t.Errorf で報告する** だけです。
>
> 🔍 **なぜ assert を入れなかったの?** Go チームの見解は「assert は
> 『期待と違った』としか言わないテストを量産する。**失敗時に何がどう違ったかを
> 人間が読める文で報告させたい**」です(pytest は assert 文を魔改造して
> 差分を出しますが、あれは相当な黒魔術です)。また `t.Errorf` は失敗しても
> **テストを続行** します(`t.Fatalf` は即中断)。「1 回の実行でできるだけ多くの
> 失敗を報告する」思想です。とはいえ実務では差分表示に `go-cmp`、
> assert 風に書ける `testify` が広く使われています。まず標準で書けるように
> なってから導入を検討しましょう。

## テーブル駆動テスト — Go の看板文化

同じ関数に入力パターンを何個も試したいとき、Go では
**「ケースの表 + 1 つのループ」** で書きます。これがテーブル駆動テストです。

```go
func TestCalc(t *testing.T) {
	tests := []struct {
		name   string
		weight float64
		want   int
	}{
		{"軽い荷物", 0.4, 508},
		{"標準", 3.2, 564},
		{"重量物", 42.0, 1340},
		{"ゼロ", 0, 500},
	}

	for _, tt := range tests {
		t.Run(tt.name, func(t *testing.T) { // サブテスト: 個別に実行・報告される
			got := Calc(tt.weight)
			if got != tt.want {
				t.Errorf("Calc(%v) = %d; want %d", tt.weight, got, tt.want)
			}
		})
	}
}
```

```bash
go test -run 'TestCalc/重量物' ./fare  # 表の 1 行だけ実行できる
```

ケース追加は表に 1 行足すだけ。pytest の `@pytest.mark.parametrize` と
同じ発想ですが、Go ではデコレータではなく **無名 struct のスライス** という
これまでの知識だけで組み立てられているのに注目してください。

## エラーのテスト

```go
func TestFindFare_UnknownDest(t *testing.T) {
	_, err := FindFare("atlantis")
	if !errors.Is(err, ErrNoRoute) { // 第10章の道具がここで活きる
		t.Errorf("err = %v; want ErrNoRoute", err)
	}
}
```

pytest の `with pytest.raises(...)` に相当する儀式はありません。
エラーは値なので、**普通に受け取って普通に検査する** だけです。

## HTTP ハンドラのテスト — httptest

前章のサーバーは、実際にポートを開かなくてもテストできます。

```go
import "net/http/httptest"

func TestHandlePickup(t *testing.T) {
	s := &Server{parcels: make(map[string]PickupRequest)}

	body := strings.NewReader(`{"customer":"魔法薬店","dest":"north","weight_kg":3.2}`)
	req := httptest.NewRequest("POST", "/pickup", body)
	rec := httptest.NewRecorder() // ResponseWriter のふりをする録音機

	s.handlePickup(rec, req)

	if rec.Code != http.StatusCreated {
		t.Fatalf("status = %d; want 201", rec.Code) // 前提が崩れたら Fatal で中断
	}
	if !strings.Contains(rec.Body.String(), "GX-0001") {
		t.Errorf("body = %s; want id GX-0001", rec.Body.String())
	}
}
```

`httptest.NewRecorder` が使えるのは、ハンドラが具体的なネットワークではなく
**`http.ResponseWriter` インターフェース** に書いているからです。
第9章の「小さなインターフェースが差し替えを可能にする」の実践例であり、
Go でモックライブラリの出番が少ない理由でもあります。
**テストしにくい = インターフェースの切り方が悪い** シグナル、と考えます。

## ベンチマークとカバレッジ — 標準装備の計測器

```go
func BenchmarkCalc(b *testing.B) {
	for b.Loop() { // Go 1.24+ (それ以前は for i := 0; i < b.N; i++)
		Calc(3.2)
	}
}
```

```bash
go test -bench=. ./fare        # ns/op(1 回あたりのナノ秒)が出る
go test -cover ./...           # カバレッジ
go test -race ./...            # 前章のレース検出もテストで常用
go test -coverprofile=c.out ./... && go tool cover -html=c.out  # 行単位で可視化
```

Python では timeit / coverage.py / pytest-benchmark と別々に入れていた道具が、
`go test` 1 つに同居しています。

## 外部公開しないテスト補助 — 練習でよく迷う点

- テストは同じパッケージに置くので、**非公開関数(小文字)もテストできます**
- あえて `package fare_test` として置くと「利用者と同じ目線」の外部テストになります
  (公開 API の使い勝手検証に使う、一歩進んだ技です)
- テストデータのファイルは `testdata/` ディレクトリに置きます(go ツールが無視してくれる
  予約名です)

## 🚇 完成コード: `express/ledger/ledger_test.go`

第8章の台帳に検査を付けます。

```go
package ledger

import "testing"

func TestBookAddAndFind(t *testing.T) {
	book := NewBook()

	p := book.Add("north", 3.2)
	if p.ID != "GX-0001" {
		t.Errorf("最初の伝票番号 = %s; want GX-0001", p.ID)
	}

	found, ok := book.Find(p.ID)
	if !ok {
		t.Fatal("登録した荷物が見つからない")
	}
	if found.Dest != "north" {
		t.Errorf("Dest = %s; want north", found.Dest)
	}
}

func TestBookFind_NotFound(t *testing.T) {
	book := NewBook()
	if _, ok := book.Find("GX-9999"); ok {
		t.Error("存在しない伝票番号で ok=true が返った")
	}
}

func TestParcelFare(t *testing.T) {
	tests := []struct {
		name   string
		weight float64
		want   int
	}{
		{"最軽量", 0, 500},
		{"標準", 3.2, 564},
		{"重量物", 42.0, 1340},
	}
	for _, tt := range tests {
		t.Run(tt.name, func(t *testing.T) {
			p := &Parcel{Weight: tt.weight}
			if got := p.Fare(); got != tt.want {
				t.Errorf("Fare() = %d; want %d", got, tt.want)
			}
		})
	}
}
```

> 💡 `Find` を「カンマ ok」でなく `(*Parcel, error)` を返す設計に変えて、
> 第10章の `ErrNotFound` を `errors.Is` で検査するテストに発展させるのも
> 良い練習です(演習 1 がその入り口です)。

```bash
go test -v -race -cover ./ledger
```

## 📝 今日の配達訓練(演習)

1. 第10章の `loadCargo` にテーブル駆動テストを書いてください。表の列は
   `name / dest / weight / wantErr`(期待するエラー)です。エラー比較には
   `errors.Is` / `errors.As` を使い分けましょう。
2. わざとバグ(`weight*20` を `weight*2` に)を入れてテストが検出することを確認し、
   直してから `-cover` でカバレッジを見てください。
3. 第4章の `slices.Clone` する版としない版のベンチマークを書き、
   コピーのコストを ns/op で観察してください。

---

検査体制が整いました。いよいよ最終章。これまでの全部品——パッケージ設計、
インターフェース、エラー設計、並行処理、HTTP、テスト——を組み上げて、
**1 つのバイナリで配布できる配送管理システム** を完成させます。
→ [第16章 卒業制作](16_final.md)
