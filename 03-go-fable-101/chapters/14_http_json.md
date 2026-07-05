# 第14章 電子受付を開く — net/http と JSON

## 🚇 今日のお話

お客さんから「いちいち営業所まで来るのは面倒。Web で集荷依頼させてくれ」と
要望が来ました。今日は **HTTP サーバー** を建て、**JSON** で依頼を受け付けます。

Python なら Flask や FastAPI を pip で入れる場面ですが、Go では
**標準ライブラリだけで実用サーバーが建ちます**。これは Go が
「Google のサーバーを書くための言語」として生まれたことの直接の帰結です。

## 最小のサーバー — 30 秒で開店

```go
package main

import (
	"fmt"
	"net/http"
)

func main() {
	http.HandleFunc("GET /health", func(w http.ResponseWriter, r *http.Request) {
		fmt.Fprintln(w, "Gopher Express 営業中 🚇")
	})

	fmt.Println("http://localhost:8080 で受付開始")
	http.ListenAndServe(":8080", nil)
}
```

```bash
go run . &
curl localhost:8080/health
```

- ハンドラは `(w http.ResponseWriter, r *http.Request)` を受ける関数。
  **w に書いたものがレスポンス** になります(w は第9章の `io.Writer` です!
  だから `fmt.Fprintln` がそのまま使える——小さなインターフェースの威力です)
- `"GET /health"` のようにメソッドやパスパラメータ(`/parcels/{id}`)を
  パターンに書けるのは Go 1.22 からです。古い記事ではこれができず、
  gorilla/mux や chi などのルーターライブラリを入れていました

> 🔍 **知らないうちに並行サーバー**: `net/http` は **リクエストごとに goroutine を
> 起動** します。何も書かなくても最初からマルチコアで並行処理するサーバーです。
> Python で gunicorn のワーカー数を調整していた作業は存在しません。
> 裏返すと、**ハンドラが触る共有データには前章の知識(Mutex)が必須** です。

## encoding/json — 伝票の電子化

### struct タグ — フィールド名の対訳表

```go
type PickupRequest struct {
	Customer string  `json:"customer"`
	Dest     string  `json:"dest"`
	Weight   float64 `json:"weight_kg"`
	Note     string  `json:"note,omitempty"` // 空なら JSON から省く
}
```

バッククォートの中身は **struct タグ**。「Go では `Weight`、JSON では `weight_kg`」
という対訳表です。

> 🔍 **なぜタグなんてものが要るの?**
> 第6章の掟——「公開フィールドは大文字始まり」——を思い出してください。
> JSON の世界の慣習は `snake_case` や `camelCase` なので、Go のフィールド名を
> そのまま使えません。この対訳をアノテーションで書くための汎用機構が struct タグで、
> `json:` 以外にも `db:`、`yaml:`、バリデーションなど、エコシステム全体が
> この 1 つの仕組みに相乗りしています。実体はただの文字列で、ライブラリが
> リフレクション(実行時の型情報検査)で読み取ります。

### Marshal / Unmarshal

```go
req := PickupRequest{Customer: "魔法薬店", Dest: "north", Weight: 3.2}

data, err := json.Marshal(req) // Go → JSON([]byte)
// {"customer":"魔法薬店","dest":"north","weight_kg":3.2}

var back PickupRequest
err = json.Unmarshal(data, &back) // JSON → Go(ポインタを渡す!)
```

> 🐍 **Python との違い①: dict ではなく struct に読み込む**
> Python の `json.loads` は何でも dict にしました。Go は **先に形(struct)を
> 宣言し、そこに流し込み** ます。型のない JSON はどう受けるのかというと
> `map[string]any` も使えますが、取り出すたびに型アサーションの嵐になるので、
> 形が分かっているなら struct 一択です。

### ⚠️ JSON の落とし穴 3 連発

```go
// ① 小文字フィールドは「静かに」無視される
type bad struct {
	customer string `json:"customer"` // 非公開なので Marshal も Unmarshal も素通り
}
// エラーは出ない。JSON が {} になって初めて気づく。go vet が警告してくれる

// ② Unmarshal で「JSON にないフィールド」はゼロ値のまま
// 「weight_kg が来なかった」のか「0 だった」のか区別できない
// → 区別が必要なら *float64(ポインタ)にする。nil = 来なかった

// ③ nil スライスは null になる(第4章の伏線回収)
var items []string
json.Marshal(items)          // null
json.Marshal([]string{})     // []
// API のお客さんは null より [] を期待しがち。空で初期化して返すのが親切
```

## 受付 API 本体 — POST を受けて登録する

```go
type Server struct {
	mu      sync.Mutex
	nextID  int
	parcels map[string]PickupRequest
}

func (s *Server) handlePickup(w http.ResponseWriter, r *http.Request) {
	var req PickupRequest
	if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
		http.Error(w, "JSON が読めません: "+err.Error(), http.StatusBadRequest)
		return // ← http.Error は return しない! 忘れると処理が続いてしまう
	}
	if req.Dest == "" || req.Weight <= 0 {
		http.Error(w, "dest と weight_kg は必須です", http.StatusBadRequest)
		return
	}

	s.mu.Lock()
	s.nextID++
	id := fmt.Sprintf("GX-%04d", s.nextID)
	s.parcels[id] = req
	s.mu.Unlock()

	w.Header().Set("Content-Type", "application/json")
	w.WriteHeader(http.StatusCreated)
	json.NewEncoder(w).Encode(map[string]string{"id": id})
}
```

- `json.NewDecoder(r.Body)` はストリームから直接読む形。`io.Reader` を受けるので
  「HTTP ボディでもファイルでもテスト用文字列でも同じコード」になります
- **`http.Error` の後の `return` 忘れ** は Go の Web 開発の定番バグです。
  Python のフレームワークは例外や return で抜けるのが自然でしたが、
  Go のハンドラはただの関数——自分で return しない限り続行します
- ヘッダー → `WriteHeader`(ステータス)→ ボディ、の **順番厳守** です。
  ボディを書き始めた瞬間に 200 が確定します

## クライアント側 — 依頼を出す方も書く

```go
body, _ := json.Marshal(PickupRequest{Customer: "鍛冶屋", Dest: "south", Weight: 42})

resp, err := http.Post("http://localhost:8080/pickup", "application/json",
	bytes.NewReader(body))
if err != nil {
	return fmt.Errorf("受付への接続: %w", err)
}
defer resp.Body.Close() // 忘れると接続がリークする。Post したら即 defer

if resp.StatusCode != http.StatusCreated {
	return fmt.Errorf("受付拒否: %s", resp.Status)
}
```

> 🐍 `requests.post(url, json=payload).json()` の 1 行が 10 行になりました。
> Go の標準は requests ほど親切ではなく、**「エラーも Close もステータス確認も
> 自分でやる」** 素材です。実務では薄いヘルパーを一度書いて使い回します。
> なお `resp.Body.Close()` 忘れは静かなリソースリークの名所です。

## 🚇 完成コード: `express/day14/main.go`

```go
// Gopher Express — 電子受付所
package main

import (
	"encoding/json"
	"fmt"
	"net/http"
	"sync"
)

type PickupRequest struct {
	Customer string  `json:"customer"`
	Dest     string  `json:"dest"`
	Weight   float64 `json:"weight_kg"`
}

type Server struct {
	mu      sync.Mutex
	nextID  int
	parcels map[string]PickupRequest
}

func (s *Server) handlePickup(w http.ResponseWriter, r *http.Request) {
	var req PickupRequest
	if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
		http.Error(w, `{"error":"invalid json"}`, http.StatusBadRequest)
		return
	}
	if req.Dest == "" || req.Weight <= 0 {
		http.Error(w, `{"error":"dest and weight_kg required"}`, http.StatusBadRequest)
		return
	}

	s.mu.Lock()
	s.nextID++
	id := fmt.Sprintf("GX-%04d", s.nextID)
	s.parcels[id] = req
	s.mu.Unlock()

	w.Header().Set("Content-Type", "application/json")
	w.WriteHeader(http.StatusCreated)
	json.NewEncoder(w).Encode(map[string]string{"id": id})
}

func (s *Server) handleGet(w http.ResponseWriter, r *http.Request) {
	id := r.PathValue("id") // Go 1.22+ のパスパラメータ
	s.mu.Lock()
	p, ok := s.parcels[id]
	s.mu.Unlock()
	if !ok {
		http.Error(w, `{"error":"not found"}`, http.StatusNotFound)
		return
	}
	w.Header().Set("Content-Type", "application/json")
	json.NewEncoder(w).Encode(p)
}

func main() {
	s := &Server{parcels: make(map[string]PickupRequest)}

	mux := http.NewServeMux()
	mux.HandleFunc("POST /pickup", s.handlePickup)
	mux.HandleFunc("GET /parcels/{id}", s.handleGet)

	fmt.Println("🚇 電子受付 http://localhost:8080 で開始")
	if err := http.ListenAndServe(":8080", mux); err != nil {
		fmt.Println("受付終了:", err)
	}
}
```

```bash
go run . &
curl -X POST localhost:8080/pickup \
  -d '{"customer":"魔法薬店","dest":"north","weight_kg":3.2}'
# {"id":"GX-0001"}
curl localhost:8080/parcels/GX-0001
# {"customer":"魔法薬店","dest":"north","weight_kg":3.2}
```

## 📝 今日の配達訓練(演習)

1. `GET /parcels`(全件一覧)を追加してください。map をスライスに詰め替えて返すとき、
   0 件時に `null` ではなく `[]` を返す工夫(落とし穴③)を入れましょう。
2. Mutex を外して `for i in $(seq 100); do curl -s -X POST ... & done` のような
   並列リクエストを浴びせ、`go run -race` で競合レポートを見てください。
3. `Weight` を `*float64` に変え、「フィールド未指定」と「0 指定」を
   区別できることを確認してください(落とし穴②)。

---

受付は開きましたが、「動いてるはず」で運用はできません。Gopher Express は
すべての設備に **出荷前検査** を義務付けます。Go のテストは pytest とかなり
気配が違います——テーブル駆動テストという Go 独自の文化へ。
→ [第15章 出荷前検査](15_testing.md)
