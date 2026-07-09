# 第5章 工房の秘伝 — メタプログラミングと DSL の正体

## 💎 今日のお話

夜、親方が蔵から古い巻物を取り出しました。「Ruby 職人の秘伝を教える。
**コードを書くコード** ——これを知る者だけが、鉄道(Rails)の設計図を読める」

この章のゴールはただ 1 つ。Rails 教材で毎日目にすることになるこの 1 行を、

```ruby
validates :name, presence: true, length: { maximum: 100 }
```

**文法的に完全に説明できるようになる** ことです。部品は 4 つ。
順番に分解していきましょう。

## 部品① クラス定義の中身は「実行されるコード」

Python でも Go でも、クラスや struct の定義は「宣言」でした。
Ruby では違います。**class ... end の中身は、上から順に実行される普通のコード** です。

```ruby
class Jewel
  puts "クラスを定義中です!"   # ← 定義した瞬間に表示される!

  3.times { |i| puts "研磨 #{i}" }  # ループだって回せる

  if ENV["LUXURY_MODE"]             # 条件分岐でメソッド定義を変えることも
    def price = 100_000
  else
    def price = 4_800
  end
end
```

つまり前章の `attr_accessor :name` は、「クラス定義を実行している最中に
呼ばれたメソッド」だったのです。では誰のメソッドか? クラス定義の中では
`self` はそのクラス自身(`Jewel`)です。`attr_accessor` は
**クラスというオブジェクトに対するメソッド呼び出し** でした。

このような「クラス定義の中で呼んで、クラスの構造を変えるメソッド」を
**クラスマクロ** と呼びます。`attr_accessor` も、Rails の `validates` も
`has_many` も、全部クラスマクロです。

## 部品② define_method — メソッドを作るメソッド

クラスマクロは自作できます。`attr_accessor` を自分の手で再発明してみましょう。

```ruby
class Jewel
  def self.my_attr_accessor(*names)      # *names は可変長引数(Python の *args)
    names.each do |name|
      define_method(name) do             # ゲッターを動的に定義
        instance_variable_get("@#{name}")
      end
      define_method("#{name}=") do |value|  # セッターを動的に定義
        instance_variable_set("@#{name}", value)
      end
    end
  end

  my_attr_accessor :name, :price         # 自作クラスマクロを使う!
end

j = Jewel.new
j.name = "garnet"    # 動く!このメソッドは実行時に「生成」された
puts j.name
```

`define_method(名前) do ... end` は「その名前のメソッドを、ブロックを本体として
定義する」——**メソッド名を文字列やシンボルで組み立てられる** のが肝です。
Rails が `has_many :orders` の 1 行から `orders`、`orders=`、`order_ids` など
十数個のメソッドを生やすのは、内部でこれをやっているからです(第11章で対面します)。

> 🐍 **Python との違い①: 秘伝が「表芸」である**
> Python にも `setattr` やメタクラス、デコレータがあり、同じことは可能です
> (Python 教材の最終盤でやりました)。違いは **文化的な位置づけ** です。
> Python ではメタプログラミングは「フレームワーク作者の裏芸」で、
> 日常コードに書くと眉をひそめられます。Ruby では `attr_accessor` という
> 形で **入門者が最初の週に使う表芸** です。言語の中心にメタプログラミングが
> 座っている——これが「Ruby は DSL の女王」と呼ばれる理由の核心です。

## 部品③ 括弧の省略と「最後の引数のハッシュ」

第2章でメソッド呼び出しの括弧は省略できると学びました。もう 1 つの重要ルール:
**引数の最後がハッシュのとき、`{ }` を省略できます**。

```ruby
def register(name, options)
  puts "#{name} を登録(オプション: #{options})"
end

register("garnet", { color: "red", grade: "A" })   # 丁寧に書くと
register("garnet", color: "red", grade: "A")       # {} を省略
register "garnet", color: "red", grade: "A"        # () も省略
```

3 行目を見てください。**もはや設定ファイルのように見えます**。でも正体は
ただのメソッド呼び出しです。ここまでの知識で、あの 1 行が読めるようになりました:

```ruby
validates :name, presence: true, length: { maximum: 100 }
# ↓ 省略を全部戻すと
validates(:name, { presence: true, length: { maximum: 100 } })
# = クラスマクロ validates に、シンボル 1 個とハッシュ 1 個を渡している。それだけ!
```

キーワード引数も同じ見た目です(受け側の定義が違うだけ):

```ruby
def price_tag(base, tax:, discount: false)   # tax: は必須キーワード引数
  ...
end
price_tag(4800, tax: 0.1)
```

## 部品④ send と method_missing — 動的ディスパッチの奥の手

**send** は「メソッド名を実行時に決めて呼ぶ」道具です。

```ruby
jewel.send(:name)          # jewel.name と同じ
column = :price
jewel.send(column)         # 変数に入ったメソッド名で呼べる
jewel.send(:secret_cost)   # private メソッドも呼べてしまう(テストで時々使う)
```

**method_missing** はさらに強力で、「存在しないメソッドが呼ばれたときに
最後に発動するフック」です。

```ruby
class Catalog
  def initialize(data) = @data = data

  def method_missing(name, *args)
    if @data.key?(name)
      @data[name]                # データにあれば、その値を返す
    else
      super                      # なければ通常どおり NoMethodError へ
    end
  end

  def respond_to_missing?(name, include_private = false)
    @data.key?(name) || super    # method_missing とセットで定義するのが作法
  end
end

c = Catalog.new(garnet: 4800, ruby: 98000)
c.garnet    # => 4800   存在しないメソッドなのに動く!
c.diamond   # => NoMethodError(super に流れた)
```

かつての Rails は `find_by_name_and_price(...)` のような「呼ばれたメソッド名を
解析して SQL を組み立てる」芸当を method_missing でやっていました。
強力ですが、乱用すると **「どこにも定義がないのに動くコード」** が生まれ、
追跡が困難になります。現代の Rails は method_missing への依存を減らし、
define_method による事前生成へ移行してきました——それでも、フレームワークの
足元では今も動いている技術です。

## 総仕上げ — ミニ validates を自作する

4 つの部品を組み合わせ、Rails 風の DSL を **自分の手で** 作ります。
これができれば、Rails 編で魔法に見えるものは何もなくなります。

```ruby
module MiniValidatable
  def self.included(base)        # include されたときに発動するフック
    base.extend ClassMethods     # クラスマクロをクラス側に注入
  end

  module ClassMethods
    def validates(attr, presence: false, numericality: false)
      @rules ||= []
      @rules << { attr:, presence:, numericality: }   # キーの省略記法(Ruby 3.1+)
    end

    def rules = @rules || []
  end

  def valid?
    self.class.rules.all? do |rule|
      value = send(rule[:attr])                        # 属性値を動的に取得
      ok = true
      ok &&= !value.nil? && value != "" if rule[:presence]
      ok &&= value.is_a?(Numeric)       if rule[:numericality]
      ok
    end
  end
end

class Jewel
  include MiniValidatable
  attr_accessor :name, :price

  validates :name,  presence: true          # ← 自作 DSL!Rails と同じ見た目
  validates :price, presence: true, numericality: true
end

j = Jewel.new
j.valid?              # => false(name も price も空)
j.name  = "garnet"
j.price = 4800
j.valid?              # => true
```

流れを言葉にすると——`include MiniValidatable` の瞬間に `included` フックが走り、
`validates` というクラスマクロが `Jewel` に生える。クラス定義の実行中に
`validates :name, presence: true` が呼ばれ、ルールがクラスに蓄積される。
インスタンスの `valid?` は、蓄積されたルールを `send` で検証する。

**Rails の ActiveModel はこれの豪華版です。** 構造はまったく同じです。

> 🔍 **なぜそうなっているの? — Ruby の強みは「言語内言語」を作る力**
> Python は「誰が書いても同じになる 1 つの正しい書き方」を、Go は
> 「機能を削って読み手を守ること」を選びました。Ruby が選んだのは
> **「問題領域ごとに、その領域の言葉で書ける小さな言語(DSL)を作る」** 力です。
> 括弧の省略・ブロック・オープンクラス・クラスマクロ・method_missing——
> 単体では小さな柔軟性が、組み合わさると「Web アプリの語彙(validates,
> has_many, resources)」「テストの語彙(describe, it, expect)」
> 「インフラの語彙(Chef の recipe)」を生み出します。
> Rails、RSpec、Chef、Vagrant、Homebrew——Ruby 製の道具が軒並み
> 「設定ファイルに見えるコード」を持つのは偶然ではありません。
> **DSL こそが Ruby の主戦場** であり、Rails はその最高傑作です。

> 🐹 **Go との違い①: 対極の哲学を確認する**
> Go にはこの章の機能が **1 つもありません**。実行時のメソッド定義も、
> 括弧の省略も、オープンクラスも、意図的に排除されています(リフレクションは
> ありますが日常的には使いません)。Go のコードは「書いてあることがすべて」で、
> 定義へのジャンプが必ず成功します。Ruby のコードは「実行してみるまで
> 何が生えているか分からない」代わりに、圧倒的に少ない記述量で意図を表せます。
> **grep しやすさの Go、書き心地の Ruby。** どちらが優れているかではなく、
> 何と引き換えに何を得たか——3 言語目のあなたなら、もう比べられるはずです。

## 💎 完成コード: `atelier/day5.rb`

上の「ミニ validates」のコード全体を `atelier/day5.rb` として保存し、
末尾に動作確認を足して実行してください:

```ruby
# (MiniValidatable と Jewel の定義は本文のとおり)

checks = [
  Jewel.new.tap { |j| j.name = "garnet"; j.price = 4800 },
  Jewel.new.tap { |j| j.name = "" },
  Jewel.new.tap { |j| j.name = "ruby"; j.price = "高い" },
]

checks.each_with_index do |j, i|
  puts "石#{i + 1}: #{j.valid? ? '✅ 合格' : '❌ 不合格'}(name=#{j.name.inspect}, price=#{j.price.inspect})"
end
```

`tap` は「オブジェクトにブロックを適用して、オブジェクト自身を返す」便利メソッドです。
これも「ブロックがあるから作れた」道具の 1 つです。

## 📝 今日の研磨(演習)

1. ミニ validates に `length:` オプション(`length: { maximum: 10 }` の形)を
   追加してください。ネストしたハッシュの読み方の練習です。
2. `Catalog` の method_missing 例で、`respond_to_missing?` を **コメントアウト** して
   `c.respond_to?(:garnet)` を試してください。false になる(=嘘をつく)ことを確認し、
   なぜセット定義が作法なのか考えましょう。
3. **総仕上げの確認テスト:** 次の Rails コードを、省略をすべて戻した形
   (括弧・波括弧付き)に書き直してください。
   ```ruby
   has_many :orders, dependent: :destroy
   scope :in_stock, -> { where("stock > 0") }
   ```
   (答え合わせ: `has_many(:orders, { dependent: :destroy })` /
   `scope(:in_stock, lambda { where("stock > 0") })` —— 全部ただのメソッド呼び出しです)

---

秘伝の巻物は読み終えました。翌朝、町に汽笛が響きます。
鉄道の開通です。親方が言いました。「紅玉堂も、時代に乗るぞ。**通販を始める**」
いよいよ Ruby on Rails の世界へ。
→ [第6章 鉄道開通](06_hello_rails.md)
