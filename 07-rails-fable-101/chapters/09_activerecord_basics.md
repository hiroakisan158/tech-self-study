# 第9章 商品台帳 — モデルと ActiveRecord

## 🚂 今日のお話

紅玉堂の商品台帳は、代々手書きの和紙でした。「これをデータベースに移す」と親方。
「だが SQL を 1 行も書かずにやる。**台帳係(モデル)に日本語で頼めば、
台帳係が SQL に訳してくれる**——それが ActiveRecord だ」

この章は Rails の心臓部です。そして第5章の秘伝が最も派手に働く場所でもあります。

## モデルを生成する

```bash
bin/rails generate model Jewel name:string stone:string price:integer stock:integer
```

2 つのファイルが生まれます(テストも生成されますが第14章で扱います):

```ruby
# db/migrate/20260709000001_create_jewels.rb — マイグレーション
class CreateJewels < ActiveRecord::Migration[8.0]
  def change
    create_table :jewels do |t|
      t.string  :name
      t.string  :stone
      t.integer :price
      t.integer :stock

      t.timestamps    # created_at / updated_at を自動追加
    end
  end
end
```

```ruby
# app/models/jewel.rb — モデル本体
class Jewel < ApplicationRecord
end
```

> 💡 **なぜモデル名を `Gem` にしなかったのか。** Ruby には標準で `Gem` という
> モジュール(RubyGems 本体)が存在するため、`Gem` というモデルを作ると
> 名前が衝突して大事故になります。Ruby/Rails では **既存の定数名との衝突**
> に注意が必要です(`Class`、`Method`、`Application` なども危険な名前です)。

## マイグレーション — 台帳の改版履歴

マイグレーションは「テーブル定義の変更を、Ruby のコードとして履歴管理する」仕組みです。

```bash
bin/rails db:migrate     # 未適用のマイグレーションを実行
bin/rails db:rollback    # 直前の1つを取り消す
```

適用結果は `db/schema.rb` に「現在の全テーブル定義」として自動記録されます。

> 🐹 **Go との違い①: マイグレーションが「最初から入っている」**
> Go では golang-migrate や Atlas を **選んで** 導入し、SQL ファイルを
> up/down のペアで書くのが典型でした。Rails はフレームワークに標準装備で、
> しかも `change` メソッド 1 つ書けば **逆操作(rollback)を自動導出** します
> (`create_table` の逆は `drop_table`、と Rails が知っているからです)。
> 「みんなが必要とするものは全部入り」——omakase の思想がここでも一貫しています。

後からカラムを足すときも、マイグレーションを作ります(schema.rb を直接編集しません):

```bash
bin/rails generate migration AddDescriptionToJewels description:text
bin/rails db:migrate
```

コマンド名の英語 `AddDescriptionToJewels` を Rails が解析して、
`add_column :jewels, :description, :text` を **自動生成** します。命名規約が
コードを書いてくれる——徹底しています。

## ActiveRecord — 台帳係との会話

`bin/rails console` を開いてください。ここからが本番です。

```ruby
# 作成 — INSERT 文が発行される
Jewel.create(name: "紅玉の指輪", stone: "ruby", price: 98000, stock: 2)
Jewel.create(name: "柘榴石のブローチ", stone: "garnet", price: 4800, stock: 10)

# 取得 — SELECT 文が発行される
Jewel.count                 # => 2
Jewel.all                   # 全件
Jewel.find(1)               # 主キーで1件(なければ RecordNotFound 例外)
Jewel.find_by(stone: "ruby")   # 条件で1件(なければ nil)
Jewel.where("price < ?", 10000)  # 条件で複数件
Jewel.order(price: :desc).first  # 最高額の品

# 更新
ring = Jewel.find(1)
ring.update(price: 105000)   # UPDATE 文
ring.stock -= 1
ring.save                    # 変更をまとめて UPDATE

# 削除
Jewel.find(2).destroy        # DELETE 文
```

コンソールには発行された SQL がそのまま表示されます。**必ず目で追ってください**:

```
Jewel Load (0.3ms)  SELECT "jewels".* FROM "jewels" WHERE "jewels"."stone" = ? LIMIT 1
```

「Ruby のメソッド呼び出しが、どんな SQL に訳されたか」を常に意識するのが、
ActiveRecord と長く付き合う唯一のコツです(サボると第13章の N+1 で泣きます)。

## ここで第5章の伏線が全回収される

`app/models/jewel.rb` を見返してください。**中身は空** です:

```ruby
class Jewel < ApplicationRecord
end
```

なのに `jewel.name` も `jewel.price=` も動きます。`attr_accessor` すら
書いていないのに、です。種明かし——ActiveRecord は起動時に
**テーブル定義(jewels テーブルのカラム一覧)を DB から読み取り、
カラムごとにアクセサメソッドを動的に定義** しています。第5章で自作した
`my_attr_accessor` の、DB 連動版です。

- テーブルにカラムを足せば、モデルにメソッドが生える(DRY — 定義は DB に 1 箇所)
- `Jewel` というクラス名から `jewels` というテーブル名を **複数形変換** で導出
  (`Person` → `people` のような不規則変化まで組み込みの辞書で対応します。
  第6章の演習で見た `"jewel".pluralize` はこのための道具でした)

> 🔍 **なぜそうなっているの? — Active Record パターンと Rails の賭け**
> 「テーブルの 1 行 = オブジェクト 1 個。行の CRUD をオブジェクト自身の
> メソッドにする」——これは Martin Fowler が 2003 年の著書で整理した
> **Active Record パターン** で、Rails はフレームワークの中核にこれを据えました。
> 対抗馬は **Data Mapper パターン**(オブジェクトと DB を分離し、写像係を挟む。
> Python の SQLAlchemy、Java の Hibernate がこちら)です。
> Active Record は「モデル = テーブル」の対応が保てる **中小規模では圧倒的に速く
> 書け**、対応が崩れる複雑なドメインでは苦しくなります。Rails はここでも
> 「大多数のアプリは CRUD が主戦場」に賭けて、シンプルな方を選びました。
> なお Python の Django ORM も Active Record 型です——Rails の成功を見て
> 採用した設計で、ここでも Rails は「上流」にいます。

> 🐍 **Python との違い①: モデル定義に属性を書かない**
> Django では `name = models.CharField(max_length=100)` とモデルに属性を
> **明示的に列挙** し、マイグレーションはそこから自動生成されました
> (コードが真実の源)。Rails は逆で、**DB スキーマが真実の源** であり、
> モデルには何も書きません。「モデルを開いてもカラムが分からない」のは
> Rails の有名な弱点で、実務では annotate という gem でコメントを
> 自動追記したり、schema.rb を隣に開いたりして補います。

## seeds — 開発用の初期データ

毎回コンソールでデータを作るのは面倒です。`db/seeds.rb` に書いておきましょう:

```ruby
# db/seeds.rb
Jewel.destroy_all

[
  { name: "紅玉の指輪",       stone: "ruby",     price: 98000, stock: 2 },
  { name: "柘榴石のブローチ", stone: "garnet",   price: 4800,  stock: 10 },
  { name: "蒼玉のペンダント", stone: "sapphire", price: 45000, stock: 5 },
  { name: "水晶の風鈴",       stone: "quartz",   price: 1200,  stock: 0 },
].each { |attrs| Jewel.create!(attrs) }

puts "🌱 #{Jewel.count} 点の商品を登録しました"
```

```bash
bin/rails db:seed
```

`create` ではなく `create!` を使っています。`!` 版は失敗時に **例外を投げる** ので、
seed のような「失敗に気づきたい」場面で使います(第2章の `!` の慣習が
Rails では「静かに false を返す版 / 例外を投げる版」の使い分けになっています。
詳しくは次章)。

## コントローラを本物の台帳につなぐ

仮データのハッシュに別れを告げます:

```ruby
# app/controllers/jewels_controller.rb
class JewelsController < ApplicationController
  def index
    @jewels = Jewel.order(:price)
  end

  def show
    @jewel = Jewel.find(params[:id])
  end
end
```

```erb
<%# app/views/jewels/index.html.erb %>
<h1>💎 商品一覧</h1>
<% @jewels.each do |jewel| %>
  <div class="jewel-card">
    <h2><%= link_to jewel.name, jewel_path(jewel) %></h2>
    <p><%= number_to_currency(jewel.price, unit: "¥", precision: 0) %></p>
    <%= stock_badge(jewel.stock) %>
  </div>
<% end %>
```

`jewel_path(jewel)` にモデルをそのまま渡せている点、`jewel[:name]` ではなく
`jewel.name` とメソッドで呼べている点——ハッシュ時代との違いを味わってください。

## 📝 今日の研磨(演習)

1. コンソールで `Jewel.where("stock > 0").order(price: :desc).map(&:name)` を実行し、
   発行された SQL を書き写してください。Ruby のチェーンと SQL の対応を
   自分の手で確認するのが目的です。
2. **壊す実験:** `Jewel.find(9999)` と `Jewel.find_by(id: 9999)` を両方実行し、
   例外と nil という **失敗の流儀の違い** を確認してください。ブラウザで
   `/jewels/9999` を開くと、開発環境では RecordNotFound のエラー画面、
   本番設定では自動的に 404 になります(Rails が例外を HTTP ステータスに
   変換しています)。
3. マイグレーションで `jewels` に `description:text` を追加し、
   **モデルを 1 文字も変更せずに** `jewel.description` が使えることを
   確認してください。ActiveRecord の動的定義を体感する演習です。

---

台帳はデータベースになりました。しかし今は「価格がマイナスの商品」も
「名前のない商品」も登録できてしまいます。台帳に嘘を書かせない仕組み——
検品(バリデーション)を導入します。
→ [第10章 検品](10_validations.md)
