# 第13章 在庫検索 — クエリ、scope、そして N+1

## 🚂 今日のお話

商品が 300 点を超え、「赤い石で、在庫があって、5 万円以下のもの」といった
問い合わせが増えてきました。台帳係(ActiveRecord)への頼み方を
一段深く学ぶ日です。

そして今日は、Rails の代名詞とまで言われる病 —— **N+1 問題** を
自分の目で発症させ、自分の手で治します。

## クエリインターフェース — つないで、絞って、並べる

```ruby
Jewel.where(stone: "ruby")                    # WHERE stone = 'ruby'
Jewel.where("price <= ?", 50_000)             # プレースホルダ(SQLインジェクション防止)
Jewel.where(stone: ["ruby", "garnet"])        # IN 句に展開される
Jewel.where.not(stock: 0)                     # WHERE stock != 0
Jewel.order(price: :desc).limit(10)           # 上位10件
Jewel.where(stone: "ruby").or(Jewel.where(stone: "garnet"))

Jewel.pluck(:name)          # SELECT name だけ(モデル化せず値の配列で返す。軽い)
Jewel.exists?(stone: "opal")  # 存在チェック(SELECT 1 ... LIMIT 1)
Jewel.group(:stone).count     # => {"ruby"=>12, "garnet"=>34, ...}
Jewel.sum(:price)             # DB 側で集計(Ruby 側で sum するより遥かに速い)
```

`?` プレースホルダは必ず使ってください。`where("name = '#{params[:q]}'")` のような
文字列連結は **SQL インジェクションの入口** です(Go で `database/sql` の
プレースホルダを使ったのと同じ理由です)。

## 遅延評価 — SQL はまだ発行されていない

ここは ActiveRecord 理解の要所です。

```ruby
query = Jewel.where(stone: "ruby")   # ← この時点で SQL は発行されていない!
query = query.where("price < ?", 100_000)  # 条件を積み増し(まだ発行されない)
query = query.order(:price)                # まだ

query.each { |j| puts j.name }       # ← ここで初めて1本の SELECT が飛ぶ
```

`where` が返すのは結果の配列ではなく **ActiveRecord::Relation**(クエリの設計図)
です。設計図は何度でも継ぎ足せて、**実際に中身が必要になった瞬間**(each、first、
count、to_a…)に 1 本の SQL へ合成されます。

> 🐍 **Python との違い①: ジェネレータの遅延と同じ発想**
> Python 教材でジェネレータの遅延評価を学びました。「必要になるまで計算しない」
> という同じ原理が、ここでは「必要になるまで SQL を発行しない」として
> 現れています。Django の QuerySet もまったく同じ設計です。
> 遅延のおかげで「条件をメソッドチェーンで組み立てる」スタイルが成立します——
> もし where のたびに SQL が飛んでいたら、チェーンは無駄撃ちの山です。

## scope — 検索条件に名前を付ける

よく使う条件は、モデルに **名前付きの部品** として登録できます:

```ruby
class Jewel < ApplicationRecord
  scope :in_stock,   -> { where("stock > 0") }
  scope :affordable, ->(limit = 50_000) { where("price <= ?", limit) }
  scope :of_stone,   ->(stone) { where(stone: stone) }
end

Jewel.in_stock.affordable.of_stone("garnet").order(:price)
```

`->` はラムダ(第3章)、`scope` はクラスマクロ(第5章)——読めますね。
scope 同士は自由に **合成** できます。Relation が設計図だからこそ、
部品を後から組み合わせて 1 本の SQL にできるのです。

コントローラの検索処理はこう書けます:

```ruby
def index
  @jewels = Jewel.in_stock.order(:price)
  @jewels = @jewels.of_stone(params[:stone]) if params[:stone].present?
end
```

## N+1 問題 — Rails の代名詞となった病

注文一覧に「顧客名」を表示する、ありふれたページを作ったとします:

```ruby
# コントローラ
@orders = Order.all
```

```erb
<% @orders.each do |order| %>
  <li><%= order.customer.name %> 様 — <%= order.total_price %> 円</li>
<% end %>
```

一見無害です。しかしログを見ると惨状が広がっています:

```
SELECT "orders".* FROM "orders"                            ← 1 回(注文一覧)
SELECT "customers".* FROM "customers" WHERE "id" = 1       ← 注文の数だけ
SELECT "customers".* FROM "customers" WHERE "id" = 2       ←   1 回ずつ
SELECT "customers".* FROM "customers" WHERE "id" = 3       ←     発行される…
...
```

注文が 100 件なら **1 + 100 = 101 回** の SQL。これが N+1 問題です。
ループの中で `order.customer` に触れるたび、その場で 1 本ずつ SELECT が
飛んでいたのです。開発中はデータが 3 件なので誰も気づかず、
**本番でデータが増えてから** ページが重くなる——典型的な発症経過です。

### 治療 — includes で「まとめて先読み」

```ruby
@orders = Order.includes(:customer)
```

```
SELECT "orders".* FROM "orders"                                  ← 1 回
SELECT "customers".* FROM "customers" WHERE "id" IN (1, 2, 3)   ← 1 回
```

101 回が **2 回** になりました。`includes` は「後で customer に触るから、
まとめて読んでおいて」という予告です。ネストした関連も先読みできます:

```ruby
Order.includes(:customer, line_items: :jewel)   # 明細とその商品まで一括
```

### 予防 — strict_loading で「遅延読み込みを禁止」する

「気づいたら直す」ではなく「発症したら即エラー」にもできます:

```ruby
@orders = Order.strict_loading.all
# ビューで order.customer に触れると ActiveRecord::StrictLoadingViolationError!
```

本番で遅くなる前に、開発中に爆発してくれるようになります。
検出用の gem(bullet)を開発環境に入れるのも定番です。

> 🔍 **なぜそうなっているの? — 抽象化の漏れ(Leaky Abstraction)**
> N+1 は Rails 固有のバグではなく、**「DB アクセスをオブジェクトの
> プロパティ参照に見せかける」あらゆる ORM の宿命** です(Django にも
> Hibernate にも SQLAlchemy にもあります)。`order.customer` という表記は
> 「メモリ上のオブジェクトを辿っているだけ」に見えますが、実体は
> ネットワーク越しの DB アクセスです。見た目のコストと実コストの乖離——
> これを Joel Spolsky は「**漏れのある抽象化の法則**」と呼びました。
> どんなに良い抽象化も、性能の話になると下の層が漏れ出してくる。
> だから「ActiveRecord を使うなら SQL ログを読め」なのです。
> Rails が悪いのではなく、**楽をした場所を覚えておく** のが使い手の責任です。

> 🐹 **Go との違い①: N+1 が「書けない」言語設計**
> Go で注文一覧 + 顧客名を作るなら、最初から JOIN 入りの SQL を書くか、
> ID を集めて IN 句で 2 本目を撃つコードを **自分の手で** 書いたはずです。
> つまり Go では N+1 は「うっかり」ではなく「わざわざ」書かないと発生しません。
> 抽象化が薄い言語は事故の種類が減る——ただし全部手書き。
> 抽象化が厚いフレームワークは 10 倍速く書ける——ただし請求書は後から来る。
> 両方を知って、案件ごとに選べるのが多言語学習者の強みです。

## カウンターキャッシュ — count の N+1 に

「顧客一覧に注文数を出す」のも N+1 の温床です(`customer.orders.count` が
1 人ずつ COUNT を撃つ)。定番の対策が **counter_cache**:

```ruby
class Order < ApplicationRecord
  belongs_to :customer, counter_cache: true   # customers.orders_count を自動更新
end
```

`customers` テーブルに `orders_count:integer` カラムを足しておくと、
注文の作成・削除のたびに Rails が自動でインクリメント/デクリメントします。
表示側は `customer.orders_count`(ただのカラム読み)で済みます。

## 💎 完成コード: 検索付き商品一覧

```ruby
# app/models/jewel.rb(scope を追記)
class Jewel < ApplicationRecord
  scope :in_stock,   -> { where("stock > 0") }
  scope :affordable, ->(limit = 50_000) { where("price <= ?", limit) }
  scope :of_stone,   ->(stone) { where(stone: stone) }
end
```

```ruby
# app/controllers/jewels_controller.rb(index を置き換え)
def index
  @jewels = Jewel.in_stock.order(:price)
  @jewels = @jewels.of_stone(params[:stone]) if params[:stone].present?
end
```

```erb
<%# index.html.erb に検索フォームを追記 %>
<%= form_with url: jewels_path, method: :get do |f| %>
  <%= f.select :stone, options_for_select(Jewel.distinct.pluck(:stone)),
        include_blank: "すべての石" %>
  <%= f.submit "絞り込む" %>
<% end %>
```

## 📝 今日の研磨(演習)

1. **N+1 を発症させる:** 注文一覧ページ(orders#index)を作り、
   `Order.all` のまま顧客名を表示して、ログで SQL の本数を数えてください。
   その後 `includes(:customer)` に直し、2 本になることを確認しましょう。
   **この往復を一度自分の手でやった人だけが、N+1 を見抜けるようになります。**
2. seeds を書き換えて注文を 50 件くらいに増やし(`50.times { ... }`)、
   演習 1 の before/after でページ表示時間がどう変わるか見てみましょう。
3. **壊す実験:** `Order.strict_loading.first.customer` をコンソールで実行し、
   StrictLoadingViolationError を観察してください。エラーメッセージが
   「どうすべきか」まで教えてくれることに注目です。

---

機能は揃ってきました。しかし「動いているはず」と「動いていることを確認した」の
間には深い谷があります。紅玉堂の品質保証体制——テストの章へ。
→ [第14章 出荷前検査](14_testing.md)
