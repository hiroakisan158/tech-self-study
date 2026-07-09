# 第3章 魔法の彫刻刀 — ブロックと Enumerable

## 💎 今日のお話

今日、工房に原石が 100 個届きました。1 個ずつ研磨の所作を繰り返すのは大変です。
親方が取り出したのは不思議な彫刻刀——**「所作を刃として差し込める」道具** でした。

「Ruby を選ぶ理由を 1 つだけ挙げろと言われたら、わしは **ブロック** と答える」

この章は Ruby 基礎編の心臓部です。ここを理解すると、Ruby のコードが
一気に読めるようになり、Rails の景色も変わります。

## ブロック = メソッドに渡す「処理のかたまり」

```ruby
stones = ["garnet", "ruby", "sapphire"]

stones.each do |stone|      # do ... end 形式(複数行向け)
  puts "#{stone} を研磨"
end

stones.each { |stone| puts "#{stone} を研磨" }   # { } 形式(1 行向け)
```

`each` は配列のメソッドで、`do...end` や `{...}` の部分が **ブロック** です。
`|stone|` はブロックの引数——each が要素を 1 つずつ、ここに渡してくれます。

Python の for 文と見比べてください:

```python
for stone in stones:        # Python: 構文(言語に組み込まれた文)
    print(stone)
```

```ruby
stones.each { |stone| puts stone }   # Ruby: メソッド呼び出し + ブロック
```

Python の `for` は言語の構文ですが、Ruby の `each` は **ただのメソッド** です。
「繰り返し」すら言語機能ではなくオブジェクトへのメッセージ——だから
**自分のクラスにも「独自のループ」を後から生やせます**。これが決定的な違いです。

## map / select / reject / sum — 変換と絞り込み

```ruby
prices = [4800, 12000, 98000, 3500]

prices.map    { |p| (p * 1.1).to_i }   # => [5280, 13200, 107800, 3850] 全要素を変換
prices.select { |p| p >= 10000 }       # => [12000, 98000]  条件で絞り込み
prices.reject { |p| p >= 10000 }       # => [4800, 3500]    条件で除外
prices.sum                             # => 118300
prices.max                             # => 98000
prices.sort.reverse                    # => [98000, 12000, 4800, 3500]
```

そして Ruby の真骨頂、**メソッドチェーン**:

```ruby
# 1万円以上の石に税を掛けて、高い順に上位2つ
prices.select { |p| p >= 10000 }
      .map    { |p| (p * 1.1).to_i }
      .sort
      .reverse
      .take(2)                         # => [107800, 13200]
```

処理が **左から右、上から下へ「文章のように」流れます**。

> 🐍 **Python との違い①: 内包表記 vs メソッドチェーン**
> Python では `[p * 1.1 for p in prices if p >= 10000]` と内包表記で書きました。
> 内包表記は 1 段なら簡潔ですが、変換 + 絞り込み + ソート + 先頭 N 件…と
> 重なると読みにくくなり、Python では途中変数に分けるのが作法でした。
> Ruby のチェーンは **何段重ねても読む方向が変わらない** のが強みです。
> Python の `map()`/`filter()` が「あまり使われない関数」なのに対し、
> Ruby では `map`/`select` こそが日常語です。

> 🐹 **Go との違い①: for ループ 3 連発からの解放**
> Go で同じ処理を書くと、スライスを宣言して for で回して append……を
> 3 回繰り返すことになります(ジェネリクス以後も慣用は for です)。
> Go はそれを「誰が読んでも分かる退屈さ」として **意図的に** 選びました。
> Ruby は逆に「意図(何をしたいか)だけを書き、手続き(どう回すか)を隠す」ことを
> 選びました。`select` という単語には「for で回して if で判定して詰め直す」が
> 圧縮されています。**Go は手続きが見える言語、Ruby は意図が見える言語** です。

## ブロック付きメソッドを自作する — yield

ブロックは組み込みメソッドの専売特許ではありません。`yield` で自作できます。

```ruby
def with_polishing_log
  puts "-- 研磨開始 --"
  result = yield          # ここでブロックが実行される
  puts "-- 研磨終了 --"
  result
end

with_polishing_log do
  puts "ガーネットをカット"
end
# -- 研磨開始 --
# ガーネットをカット
# -- 研磨終了 --
```

`yield` に引数を渡せば、ブロックの `| |` に届きます:

```ruby
def each_gemstone
  yield "garnet"
  yield "ruby"
  yield "sapphire"
end

each_gemstone { |s| puts "#{s} を検品" }
```

これで `each` の正体が分かりました。**each は「要素の数だけ yield するメソッド」** です。
魔法ではありません。

## リソース管理 — Python の with を「ただのメソッド」で

ブロックの威力が最もよく分かるのがこれです。ファイルを開いて、閉じ忘れない:

```ruby
File.open("kartes.txt", "w") do |f|
  f.puts "garnet: 検品済み"
end   # ブロックを抜けると自動で close される(例外が起きても!)
```

> 🐍🐹 **Python の with / Go の defer との比較**
> Python は `with open(...) as f:` という **専用構文** を言語に足し、
> `__enter__`/`__exit__` プロトコルを定義しました。Go は `defer f.Close()` を
> 手で書く規律を選びました。Ruby は **どちらも不要** です。
> 「開く→ブロックを実行→必ず閉じる」を `File.open` という普通のメソッドが
> `yield` + `ensure`(Ruby の finally)で実現しています。つまり
> **言語を拡張せずに、ライブラリが with 文相当を発明できる**。
> 「トランザクション内で実行」「一時的に設定を変えて実行」「時間を計って実行」…
> Rails にこのパターンが無数に出てくるのは(`ActiveRecord::Base.transaction do ... end`)、
> ブロックがあるからです。

## Proc とラムダ — ブロックを持ち運ぶ

ブロックはその場限りの使い捨てですが、オブジェクトにして変数に入れることもできます。

```ruby
tax = ->(price) { (price * 1.1).to_i }   # ラムダ(-> はラムダリテラル)
tax.call(1000)    # => 1100
tax.(1000)        # => 1100(糖衣構文)

double = proc { |x| x * 2 }              # Proc(ラムダの緩い版)
```

両者の細かい違い(引数チェックの厳しさ、return の挙動)は今は深追い不要です。
覚えるべきは **Rails で見かける `->` はラムダ**、ということ:

```ruby
# 第13章で書く Rails のコード(伏線)
scope :in_stock, -> { where("stock > 0") }
```

## &: 記法と it — もっと短く

「各要素のメソッドを 1 つ呼ぶだけ」のブロックには短縮記法があります。

```ruby
["garnet", "ruby"].map { |s| s.upcase }   # 丁寧に書くと
["garnet", "ruby"].map(&:upcase)          # &:メソッド名 で同じ意味
["garnet", "ruby"].map { it.upcase }      # Ruby 3.4 からは it も使える
```

`&:upcase` は「シンボル `:upcase` をブロックに変換して渡す」という意味です
(第5章のメタプログラミングで種明かしします)。Rails のコードでは
`users.map(&:name)` のような形で毎日目にします。

## ハッシュと each — Rails への最後の布石

配列と並ぶもう 1 つの主役、**ハッシュ**(Python の dict、Go の map)です。

```ruby
stone = { name: "garnet", price: 4800, stock: 3 }   # キーはシンボル(最頻出の形)

stone[:name]              # => "garnet"
stone[:weight]            # => nil(無いキーは nil。KeyError にならない!)
stone.fetch(:weight)      # => KeyError(Python の [] 相当の厳格さが欲しいとき)

stone.each do |key, value|
  puts "#{key}: #{value}"
end

stone.map { |k, v| "#{k}=#{v}" }.join(", ")  # ハッシュも map できる
```

`{ name: "garnet" }` は `{ :name => "garnet" }` の糖衣構文です。
古いコードや動的なキーでは `=>`(ハッシュロケット)も見かけます。

> 🐍 **Python との違い②: 無いキーは nil**
> Python の `d["missing"]` は `KeyError` で即死、Go の `m["missing"]` はゼロ値と
> `ok` の 2 値でした。Ruby は **黙って nil を返します**。事故りやすそうに見えますが、
> 「nil と false だけが偽」ルールと組み合わさって `if stone[:sale_price]` のような
> 自然な存在チェックが書けます。厳格にしたければ `fetch` — 使い分けです。

## 💎 完成コード: `atelier/day3.rb`

```ruby
# 紅玉堂 見習い3日目 — 入荷100個の一括検品
stones = [
  { name: "garnet",   carat: 2.5, price: 4800  },
  { name: "ruby",     carat: 1.2, price: 98000 },
  { name: "sapphire", carat: 0.8, price: 45000 },
  { name: "quartz",   carat: 5.0, price: 1200  },
]

# 1万円以上の高級石だけ、税込価格の名札を作り、高い順に表示
luxury = stones.select { |s| s[:price] >= 10_000 }
               .map    { |s| { name: s[:name], tag: (s[:price] * 1.1).to_i } }
               .sort_by { |s| -s[:tag] }

puts "=== 高級石コーナー ==="
luxury.each { |s| puts "#{s[:name]}: #{s[:tag]} 円(税込)" }

puts "在庫総額: #{stones.sum { |s| s[:price] }} 円"
puts "石の名前: #{stones.map { it[:name] }.join(' / ')}"

# ブロック付きメソッドの自作
def with_timer(label)
  start = Time.now
  result = yield
  puts "(#{label}: #{((Time.now - start) * 1000).round(2)}ms)"
  result
end

with_timer("全件検品") do
  stones.each { |s| s[:checked] = true }
end
```

## 📝 今日の研磨(演習)

1. `stones` から「1 カラット以上」かつ「5 万円未満」の石の名前だけを、
   **1 つのメソッドチェーン** で取り出してください。
2. `def three_times_retry` を自作してください: ブロックを実行し、例外が出たら
   最大 3 回までやり直す(ヒント: `begin/rescue` と `retry`。Ruby の例外は
   `begin/rescue/ensure` = Python の try/except/finally です)。
3. **壊す実験:** `stones.map(&:name)` を実行してエラーを観察してください
   (ハッシュに `name` メソッドはありません)。`&:` 記法が
   「各要素にそのメソッドを呼ぶ」ことを、エラーメッセージから逆に理解しましょう。

---

処理は書けるようになりました。しかし石のデータが「ただのハッシュ」なのが
気になります。宝石に **型とふるまい** を与える——次章はクラスとモジュール、
そして Ruby 流のオブジェクト指向設計です。
→ [第4章 宝石のカルテ](04_classes_modules.md)
