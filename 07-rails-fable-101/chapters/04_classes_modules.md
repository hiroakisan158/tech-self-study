# 第4章 宝石のカルテ — クラスとモジュール

## 💎 今日のお話

工房の棚には石ごとに「カルテ」が付いています。名前、重さ、価格、加工履歴。
そしてカルテには **その石に何ができるか**(研磨できる、鑑定書を出せる)も書いてあります。

データとふるまいを 1 枚のカルテにまとめる——**クラス** の出番です。
Python でも書きました。Ruby のクラスは似ていますが、ところどころで
「え、そんなに短く書けるの?」と驚くはずです。

## クラスの基本形

```ruby
class Jewel
  def initialize(name, price)   # コンストラクタ(Python の __init__)
    @name  = name               # @ 付きがインスタンス変数(Python の self.name)
    @price = price
  end

  def display_name
    "#{@name}(#{@price}円)"
  end
end

jewel = Jewel.new("garnet", 4800)   # new でインスタンス生成
puts jewel.display_name
```

Python との対応はほぼ 1:1 です:

| Python | Ruby |
|---|---|
| `def __init__(self, name):` | `def initialize(name)` |
| `self.name = name` | `@name = name` |
| `Jewel("garnet")` | `Jewel.new("garnet")` |
| `def method(self):` | `def method`(self は書かない) |

> 🐍 **Python との違い①: self を書かない**
> Python はメソッドの第 1 引数に `self` を明示しました(「明示は暗黙に勝る」)。
> Ruby ではメソッド内の `@name` が自動的に「自分の」インスタンス変数であり、
> `self` は省略がデフォルトです。Matz の美学は「**書かなくても分かることは
> 書かせない**」——Python と正反対の判断を下した典型例です。
> なお Ruby の `@name` は未定義でも `nil` を返すだけでエラーになりません。
> typo に気づきにくいので注意(これも第14章のテスト文化につながります)。

## attr_accessor — 3 行が 1 行になる

外から `@name` を読み書きしたいとき、素朴にはゲッターとセッターを書きます:

```ruby
class Jewel
  def name          # ゲッター
    @name
  end

  def name=(value)  # セッター(= で終わるメソッド名!)
    @name = value
  end
end
```

しかし Ruby では、誰もこう書きません。1 行で済むからです:

```ruby
class Jewel
  attr_accessor :name, :price   # 読み書き両方を自動定義
  attr_reader   :serial         # 読み取り専用
end

jewel = Jewel.new("garnet", 4800)
jewel.name           # => "garnet"     (ゲッターが呼ばれている)
jewel.price = 5200   # セッター price= が呼ばれている
```

ここで立ち止まってください。`attr_accessor` は **キーワードではありません**。
クラス定義の中で実行されている、**ただのメソッド呼び出し** です
(`attr_accessor(:name, :price)` の括弧が省略されているだけ)。

「クラス定義の中でメソッドを呼ぶと、メソッドが生える」——この不思議の種明かしは
第5章でします。今は **Rails の `has_many :orders` とまったく同じ形** をしていることだけ、
目に焼き付けてください。

## クラスメソッドと self

```ruby
class Jewel
  @@count = 0                  # クラス変数(全インスタンス共有。使用は控えめに)

  def self.from_hash(h)        # self. 付き = クラスメソッド(Python の @classmethod)
    new(h[:name], h[:price])   # クラスメソッド内の new は Jewel.new のこと
  end
end

jewel = Jewel.from_hash(name: "ruby", price: 98000)
```

`Jewel.from_hash(...)` のような **別ルートのコンストラクタ** は Ruby の定番です。
Rails でも `User.find(1)` や `Order.create!(...)` のようにクラスメソッドが主役になります。

## 継承 — そして「浅く」使う文化

```ruby
class AntiqueJewel < Jewel        # < が継承(Python の Jewel(...) 相当)
  def initialize(name, price, era)
    super(name, price)            # 親の initialize を呼ぶ
    @era = era
  end

  def display_name
    "【#{@era}】" + super         # super は「親の同名メソッド」
  end
end
```

Ruby は **単一継承** です(Python の多重継承は不可)。では複数の性質を
混ぜたいときはどうするか——ここで Ruby 独自の答え、**モジュールと mixin** が登場します。

## モジュールと mixin — 継承ではなく「能力の注入」

モジュールは「インスタンス化できない、メソッドの詰め合わせ」です。
`include` でクラスに混ぜ込みます。

```ruby
module Certifiable                 # 「鑑定書を発行できる」能力
  def certificate
    "鑑定書: #{certificate_body} は本物です"   # 混ぜ込み先のメソッドを呼べる
  end
end

module Engravable                  # 「刻印できる」能力
  def engrave(text)
    @engraving = text
    "「#{text}」と刻印しました"
  end
end

class Jewel
  include Certifiable
  include Engravable

  def certificate_body = @name     # 1行メソッドはこの短縮定義も可(Ruby 3.0+)
end

jewel = Jewel.new("ruby", 98000)
jewel.certificate
jewel.engrave("For K")
```

> 🐹 **Go との違い①: interface と mixin は逆向きの道具**
> Go の interface は「**何ができるべきか**(要件)」だけを定め、実装は各型が持ちました。
> Ruby の module は逆に「**実装そのもの**」を持ち、複数のクラスに配ります。
> Go が「共通のふるまいはコピーするか、埋め込みで合成しろ」と言うところを、
> Ruby は「共通実装を 1 箇所に書いて include しろ」と言います。
> そして「要件を満たすか」のチェックは——Ruby にはありません。
> メソッドがあれば呼べる、なければ実行時に `NoMethodError`。
> **完全なダックタイピングの世界** です(Go 教材第9章の「コンパイル時に検査される
> ダックタイピング」から、検査を取り除いたものと考えてください)。

### Comparable — mixin の威力を見る

「比較の全メソッドを、`<=>` 1 つから自動生成」する標準モジュールです。

```ruby
class Jewel
  include Comparable
  attr_reader :price

  def <=>(other)              # 宇宙船演算子: -1, 0, 1 を返す
    price <=> other.price
  end
end

a = Jewel.new("garnet", 4800)
b = Jewel.new("ruby", 98000)
a < b          # => true      <, >, <=, >=, ==, between? が全部使える!
[b, a].sort    # ソートもできる
```

`<=>` を 1 つ書くだけで比較演算子一式が手に入る——**「最小の定義から最大の能力を
引き出す」** のが Ruby の標準ライブラリの設計思想です。`each` を 1 つ書いて
`Enumerable` を include すれば、`map` も `select` も `sort_by` も全部生えてきます。

## オープンクラス — 禁断の(そして Rails を支える)力

Ruby では **定義済みのクラスを、あとから開いて書き足せます**。組み込みクラスすらも。

```ruby
class Integer          # 組み込みの Integer を「再オープン」
  def carats
    self * 0.2         # 1カラット = 0.2g として、グラム換算
  end
end

5.carats               # => 1.0
```

これは **モンキーパッチ** と呼ばれ、乱用すれば大事故のもとです
(チーム全員が知らないメソッドが Integer に生えている恐怖を想像してください)。
しかし Rails はこの力を全面的に活用しています:

```ruby
# Rails(ActiveSupport)を入れると使えるようになる表現
3.days.ago             # 3日前の時刻
1.hour + 30.minutes    # 時間の計算
"jewel".pluralize      # => "jewels"(複数形!第9章の伏線)
```

> 🔍 **なぜそうなっているの? — 自由と自己責任の言語**
> Python にも Go にも「組み込み型へのメソッド追加」はできません(Python は
> 組み込み型のパッチを意図的に禁止、Go はそもそも他パッケージの型に
> メソッドを足せません)。Ruby だけが全開放しているのは、Matz の思想が
> 「**プログラマは信頼されるべき大人である**」だからです。
> 危険な力も含めて渡し、使い方の節度はコミュニティの規範に委ねる。
> Rails の読みやすい `3.days.ago` も、無秩序なパッチ地獄も、同じ自由の両面です。
> 現代の Ruby には影響範囲を限定する `refinements` という安全版もありますが、
> 「節度あるモンキーパッチ」が今も現役の文化です。

## public / private

```ruby
class Jewel
  def display_name
    "#{@name}(#{formatted_price})"
  end

  private                      # これ以降は private メソッド

  def formatted_price
    "#{@price.to_s.reverse.scan(/\d{1,3}/).join(',').reverse}円"
  end
end

jewel.formatted_price   # => NoMethodError(外からは呼べない)
```

`private` も実はメソッドです(「これ以降の定義を private にせよ」という命令)。
なお Ruby の private は Python の `_name`(紳士協定)より強いものの、
`send` を使えば破れます(第5章)。**「型もアクセス制御も、最後は規律と
テストで守る」** のが Ruby の一貫した態度です。

## 💎 完成コード: `atelier/day4.rb`

```ruby
# 紅玉堂 見習い4日目 — 宝石カルテのクラス化
module Certifiable
  def certificate = "鑑定書: #{name} は本物です(#{self.class}判定)"
end

class Jewel
  include Comparable
  include Certifiable

  attr_reader :name, :carat
  attr_accessor :price

  def initialize(name, carat:, price:)
    @name, @carat, @price = name, carat, price
  end

  def <=>(other) = price <=> other.price

  def to_s = "#{name} #{carat}ct #{price}円"
end

class AntiqueJewel < Jewel
  def initialize(name, era:, **rest)
    super(name, **rest)
    @era = era
  end

  def to_s = "【#{@era}】" + super
end

stones = [
  Jewel.new("garnet", carat: 2.5, price: 4800),
  AntiqueJewel.new("ruby", era: "明治", carat: 1.2, price: 198000),
  Jewel.new("sapphire", carat: 0.8, price: 45000),
]

puts "=== 価格順カタログ ==="
stones.sort.reverse.each { |j| puts j }         # Comparable + to_s の合わせ技
puts stones.max.certificate                      # 最高額の石に鑑定書を
puts "最安: #{stones.min.name}"
```

## 📝 今日の研磨(演習)

1. `Jewel` に `discounted?` メソッド(価格が 1 万円未満なら true)を追加し、
   `stones.select(&:discounted?)` が動くことを確認してください。
   第3章の `&:` 記法とクラス設計がつながる瞬間です。
2. モジュール `Shippable` を作り、`ship(to)` メソッド(「◯◯へ発送しました」)を
   `Jewel` に include してください。さらに `AntiqueJewel` では `ship` を
   オーバーライドして「(割れ物注意)」を足し、`super` を使って親版を再利用しましょう。
3. **壊す実験:** `String` クラスをオープンして `def shout = upcase + "!!!"` を
   足し、`"ruby".shout` を試してください。動いたら、**なぜこれをチームの
   共有コードでやってはいけないのか** を自分の言葉で 1 文書いてみましょう。

---

`attr_accessor` がメソッドで、クラス定義の中で「メソッドを生やすメソッド」が呼べる——
この工房の秘伝を完全に理解すれば、Rails のコードはもう魔法には見えません。
Ruby 基礎編の最終章、メタプログラミングへ。
→ [第5章 工房の秘伝](05_metaprogramming.md)
