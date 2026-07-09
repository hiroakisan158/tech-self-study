# 第2章 職人の所作 — メソッドと制御フロー

## 💎 今日のお話

「昨日は原石を眺めるだけだったな。今日は **手順** を教える」と親方。
研磨、検品、値付け——工房の仕事は決まった所作の組み合わせです。
所作に名前を付けて呼び出せるようにする、それが **メソッド** です。

## メソッド定義 — return を書かない世界

```ruby
def polish(stone)
  "#{stone}(研磨済み)"
end

puts polish("garnet")   # => garnet(研磨済み)
```

Ruby のメソッドで最初に驚くのはこれです:

**最後に評価した式が、そのまま戻り値になる。`return` は書かない。**

```ruby
def price_with_tax(price)
  price * 1.1        # これが戻り値。return 不要
end

def grade(carat)
  if carat >= 3.0    # if も「式」なので、その結果が戻り値になる
    "特級"
  else
    "並"
  end
end
```

`return` は「途中で抜ける」ときだけ使います(ガード節)。

```ruby
def appraise(stone)
  return "鑑定不能" if stone.nil?   # ← ガード節 + 後置 if(後述)
  "#{stone} は本物です"
end
```

> 🐹 **Go との違い①: 文の言語 vs 式の言語**
> Go の `if` は文(statement)で、値を返しませんでした。だから
> `x = cond ? a : b` すら書けず、4 行の if 文が必要でした。
> Ruby では **ほぼすべてが式(expression)で値を返します**。`if` も `case` も
> `begin/rescue` も値を持つので、`grade = if ... end` と代入すらできます。
> Go が「書き方を 1 つに絞って読み手を守る」言語なら、Ruby は
> 「最も意図が伝わる書き方を書き手が選ぶ」言語です。自由の代償として、
> チームでスタイルを揃える文化(RuboCop という linter)が発達しました。

## 引数のデフォルト値と呼び出しの括弧

```ruby
def wrap(item, ribbon = "赤")   # デフォルト引数(Python と同じ感覚)
  "#{item} を #{ribbon} のリボンで包装"
end

wrap("ルビーの指輪")            # 括弧あり
wrap "ルビーの指輪"             # 括弧なしでも呼べる!
```

**メソッド呼び出しの括弧は省略できます。** 日常のコードでは括弧を付けるのが主流ですが、
この省略が許されているからこそ、Rails はこう書けるのです:

```ruby
validates :name, presence: true
# ↑ 実は validates(:name, { presence: true }) というメソッド呼び出し
```

括弧を外すと **メソッド呼び出しが「宣言文」のように読める**。
Ruby が DSL(ドメイン特化言語)の女王と呼ばれる理由の第一歩です(第5章で全貌が明かされます)。

## ? と ! — メソッド名が語る

Ruby ではメソッド名の末尾に `?` と `!` が使えます。これは文法ではなく **慣習** ですが、
徹底されているので実質的な仕様です。

```ruby
3.even?            # => false     ? は「真偽値を返す質問」
"".empty?          # => true
nil.nil?           # => true

s = "ruby"
s.upcase           # => "RUBY"    (s 自体は変わらない)
s.upcase!          # s を破壊的に変更。! は「危険・破壊的な版」
```

> 🐍 **Python との違い①: is_even() より even?**
> Python では `is_even()` や `isdigit()` と書きました。Ruby の `even?` は
> 英語の疑問文がそのままコードになります。`if stone.empty?` は
> 「もし石が空っぽなら」と **声に出して読める**。「コードは人間が読む文章である」
> という Ruby の美学が、識別子に `?` を許すという言語設計にまで及んでいるのです。

## 真偽値 — Python 経験者が必ず踏む罠

**Ruby で偽(falsy)なのは `nil` と `false` の 2 つだけ** です。

```ruby
puts "真!" if 0        # 真! と表示される(0 は真!!)
puts "真!" if ""       # 真! と表示される(空文字も真!!)
puts "真!" if []       # 真! と表示される(空配列も真!!)
puts "真!" if nil      # 何も表示されない
```

> 🐍 **Python との違い②: 0 も "" も [] も真**
> Python では `if items:` で「空でなければ」と書けました。Ruby で同じつもりで
> `if items` と書くと、**空配列でも真** なので必ず通ってしまいます。
> 空チェックは明示的に `if !items.empty?` や `unless items.empty?` と書いてください。
> 一方この仕様のおかげで、`if count` が「count が 0 だから偽」という
> Python の暗黙の落とし穴はなくなりました。「nil か、そうでないか」だけを
> 見ればよいのは、むしろ Go の明示性に近い安心感があります。

## 後置 if と unless — 文が英語になる

```ruby
# 通常の if
if stock == 0
  puts "在庫切れ"
end

# 後置 if(modifier if)— 1 行で書ける
puts "在庫切れ" if stock == 0

# unless = if not
puts "販売可能" unless stock == 0
```

後置形は **「主となる処理が先、条件が後」** という英語の語順です。
ガード節(`return ... if ...`)と組み合わせるのが Ruby の定番スタイルです。

`unless` に `else` を付けたり、複雑な条件で使うと途端に読みにくくなります。
**unless は単純な条件の否定だけに使う** のが職人の作法です。

## case/when — switch を超えるパターン照合

```ruby
def shipping_fee(region)
  case region
  when "town"          then 0
  when "east", "west"  then 300
  when /island/        then 1200   # 正規表現ともマッチできる!
  else 500
  end
end

def classify(carat)
  case carat
  when 0...1  then "小粒"     # 範囲(Range)ともマッチできる!
  when 1...3  then "中粒"
  else "大粒"
  end
end
```

`case/when` の正体は `===`(トリプルイコール)演算子です。`Range === 値`、`Regexp === 値`、
`Class === 値` がそれぞれ「含む?」「マッチする?」「インスタンス?」と定義されているため、
switch よりずっと表現力があります。Ruby 3.0 からは Python の match 文に相当する
`case/in`(パターンマッチ)もあります。

> 🐹 **Go との違い②: fallthrough の話**
> Go の switch は break 不要(自動 break)でしたが、Ruby の case/when も同じです。
> C 系の「break 忘れ事故」はどちらの言語にもありません。違うのは表現力で、
> Go の switch が値と型で分岐するのに対し、Ruby は `===` を再定義することで
> **任意のオブジェクトを「分岐条件」にできます**。柔軟さの Ruby、予測可能性の Go です。

## ループ — でも実は for を書かない

```ruby
i = 0
while i < 3
  puts "研磨 #{i + 1} 回目"
  i += 1                    # Ruby に ++ はない(Python と同じ)
end

3.times { |i| puts "研磨 #{i + 1} 回目" }   # ← こちらが Ruby 流
```

Ruby にも `for` 文はありますが、**実務コードではほぼ登場しません**。
配列の繰り返しは `each`、回数の繰り返しは `times`——すべて **メソッド + ブロック** で
書きます。なぜそれで足りるのか、ブロックとは何なのかは次章の主役です。

## ||= — 「なければ入れる」の慣用句

```ruby
config = nil
config ||= "デフォルト設定"   # config が nil/false のときだけ代入
puts config                    # => デフォルト設定
```

`x ||= y` は `x = x || y` の略で、Rails のコードに頻出します(メモ化パターン
`@result ||= heavy_calculation` として第9章以降で再会します)。

## 💎 完成コード: `atelier/day2.rb`

```ruby
# 紅玉堂 見習い2日目 — 検品と値付けの所作
def appraise(stone, carat)
  return "鑑定不能: 石がありません" if stone.nil?
  return "鑑定不能: カラットが不正です" unless carat.is_a?(Numeric) && carat > 0

  grade = case carat
          when 0...1 then "小粒"
          when 1...3 then "中粒"
          else "大粒"
          end
  "#{stone}: #{grade}(#{carat}ct)"
end

def price_tag(base, discount: false)   # キーワード引数(第5章で詳説)
  price = discount ? (base * 0.9).to_i : base
  "#{price} 円#{'(1割引)' if discount}"
end

puts appraise("garnet", 2.5)
puts appraise(nil, 1.0)
puts appraise("ruby", -1)
puts price_tag(4800)
puts price_tag(4800, discount: true)

stock = 0
puts "在庫切れです" if stock.zero?    # 0.zero? — 数にすら質問できる
```

## 📝 今日の研磨(演習)

1. `def stone_rank(price)` を書き、1万円未満なら `"C"`、1〜5万円なら `"B"`、
   それ以上なら `"A"` を返してください。`case/when` と Range を使い、
   **return を 1 つも書かずに** 実装しましょう。
2. **壊す実験:** `items = []` に対して `puts "空です" if items` を実行し、
   何も表示されない…と思いきや「空です」が表示されることを確認してください
   (空配列は真!)。その後 `items.empty?` を使った正しい版に直しましょう。
3. `10 / 3`、`10 / 3.0`、`10.fdiv(3)`、`10.divmod(3)` をそれぞれ irb で試し、
   Python 3 の `/` と `//` との対応表を自分のノートに作ってください。

---

`3.times { ... }` の波括弧の中身——あれは一体何者なのでしょう。
次章はいよいよ Ruby 最大の武器、**ブロック** です。Python の内包表記とも
Go の for ループとも違う、第三の道をお見せします。
→ [第3章 魔法の彫刻刀](03_blocks.md)
