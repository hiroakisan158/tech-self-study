# 第10章 検品 — バリデーションとコールバック

## 🚂 今日のお話

事件が起きました。台帳に「名前なし・価格 -500 円」の商品が紛れ込み、
危うくお客さんに 500 円払って石を送るところだったのです。

「台帳に書き込む前に、**必ず検品を通す**」と親方は宣言しました。
モデルに検品ルールを書く——バリデーションです。そして第5章で自作した
「ミニ validates」の本物と、ついに対面します。

## validates — もう読めるはずの 1 行

```ruby
# app/models/jewel.rb
class Jewel < ApplicationRecord
  validates :name,  presence: true, length: { maximum: 50 }
  validates :stone, presence: true
  validates :price, presence: true,
                    numericality: { only_integer: true, greater_than: 0 }
  validates :stock, numericality: { greater_than_or_equal_to: 0 }
end
```

第5章を終えたあなたには、これはもう魔法ではありません。
`validates` は **クラスマクロ**(クラス定義の実行中に呼ばれるただのメソッド)で、
`presence: true` は **波括弧を省略されたハッシュ引数**。ルールはクラスに蓄積され、
保存時に `send` で各属性を検証する——自作したミニ版と同じ構造の、豪華版です。

主な検証の語彙:

```ruby
validates :name,   presence: true                    # 空でない
validates :serial, uniqueness: true                  # 重複しない
validates :email,  format: { with: /\A[^@\s]+@[^@\s]+\z/ }  # 正規表現
validates :grade,  inclusion: { in: %w[A B C] }      # 選択肢の中から
validates :note,   length: { maximum: 400 }, allow_blank: true
```

## valid? と errors — 検品結果の読み方

コンソールで確かめましょう:

```ruby
jewel = Jewel.new(name: "", price: -500)
jewel.valid?            # => false(検品が走る)
jewel.errors.full_messages
# => ["Name can't be blank", "Stone can't be blank",
#     "Price must be greater than 0"]
jewel.errors[:price]    # 属性ごとに取り出すこともできる
```

`errors` は「どの属性が・なぜダメか」を溜め込むオブジェクトです。
第12章でフォームにエラーを表示するときの情報源になります。

## save と save! — 静かな失敗と、叫ぶ失敗

バリデーションは **保存しようとした瞬間** に自動で走ります。
失敗時の流儀が 2 系統あるのが Rails の重要な設計です:

| 静かに false / nil | 例外を投げる | 使いどころ |
|---|---|---|
| `save` | `save!` | フォーム処理は save(失敗を if で受けて再表示) |
| `create` | `create!` | seed やスクリプトは create!(失敗に気づきたい) |
| `update` | `update!` | 同上 |

```ruby
if jewel.save
  # 保存できた → 一覧へ
else
  # 検品落ち → jewel.errors を持ってフォームを再表示(第12章の型)
end
```

> 🐹 **Go との違い①: エラーの流儀が「値」と「例外」の二刀流**
> Go は「エラーは値、必ず if err != nil で受ける」の一本槍でした。
> Ruby は例外の言語ですが、Rails は **「想定内の失敗(検品落ち)は戻り値、
> 想定外の失敗は例外」** と使い分けます。ユーザーの入力ミスは日常なので
> `save` の false で受け、プログラマの想定が壊れた場面(seed で
> 保存に失敗など)は `save!` の例外で即死させる。Go の「エラーは値」の
> 感覚は、実は Rails の `save` の側に生きています。`!` の有無で
> 流儀を選べるのが Ruby 流です。

## uniqueness の罠 — アプリの検品だけでは防げない

```ruby
validates :serial, uniqueness: true
```

これは「保存前に SELECT で重複を探し、なければ通す」という実装です。
つまり **2 つのリクエストが同時に来ると、両方の SELECT が「重複なし」と
判断してすり抜けます**(レースコンディション)。

対策は DB 側にも制約を張ることです:

```ruby
# マイグレーション
add_index :jewels, :serial, unique: true
```

> 🔍 **なぜ両方書くの?**
> アプリ側の validates は「ユーザーへの親切なエラーメッセージ」のため、
> DB 側の unique index は「最後の砦」のため——役割が違うからです。
> 「バリデーションは UX、制約は整合性」と覚えてください。presence に対する
> `null: false`、外部キーに対する `foreign_key: true` も同じ関係です。
> ActiveRecord がどれだけ賢くても、**データ整合性の最終防衛線は DB** ——
> これは Rails 職人の共通認識です。

## normalizes — 検品の前に「ならす」

検品の前に入力を整えたいことがあります(前後の空白、全角/半角……)。
Rails 7.1 から専用の仕組みがあります:

```ruby
class Jewel < ApplicationRecord
  normalizes :name, with: ->(name) { name.strip }
end

Jewel.create!(name: "  紅玉の指輪  ").name   # => "紅玉の指輪"
```

`->` はもちろん第3章のラムダです。伏線は次々回収されていきます。

## コールバック — 便利と魔窟の分かれ道

モデルのライフサイクル(保存前後・削除前後など)に処理を差し込めます:

```ruby
class Jewel < ApplicationRecord
  before_save :assign_serial, if: -> { serial.blank? }

  private

  def assign_serial
    self.serial = "KG-#{SecureRandom.hex(4).upcase}"
  end
end
```

`before_save` / `after_create` / `before_destroy` など一式ありますが、
**コールバックは Rails の便利機能の中で最も「後で恨まれる」機能** です。

```ruby
# ❌ 悪名高い例
after_save :send_price_change_email   # 保存のたびにメール?テストでも?一括更新でも?
```

保存という素朴な操作に隠れた副作用がぶら下がっていくと、
「Jewel を save しただけでメールが飛び、在庫が動き、Slack に通知が行く」
**魔窟モデル** が生まれます。目安はこうです:

- ✅ **そのレコード自身の属性を整える** 処理(serial 採番、正規化)→ コールバック可
- ❌ **外の世界に作用する** 処理(メール、通知、他モデルの更新)→ コントローラや
  サービス層で明示的に呼ぶ

> 🐍 **Python との違い①: Django のシグナルと同じ罠**
> Django にも `post_save` シグナルがあり、まったく同じ理由で
> 「使いすぎるな」と言われ続けています。フレームワークを問わず
> 「保存に副作用を隠すと追跡不能になる」のは Web 開発の普遍的な教訓です。
> Go にはそもそもこの仕組みがなく、全部を呼び出し側に書かせます——
> 冗長ですが、魔窟は構造的に生まれません。**便利さの前借りは、
> 保守で返済する**。3 言語を見比べると、この原則がよく見えます。

## 💎 完成コード: `app/models/jewel.rb`

```ruby
class Jewel < ApplicationRecord
  normalizes :name, with: ->(name) { name.strip }

  validates :name,  presence: true, length: { maximum: 50 }
  validates :stone, presence: true
  validates :price, presence: true,
                    numericality: { only_integer: true, greater_than: 0 }
  validates :stock, numericality: { greater_than_or_equal_to: 0 }

  before_save :assign_serial, if: -> { serial.blank? }

  private

  def assign_serial
    self.serial = "KG-#{SecureRandom.hex(4).upcase}"
  end
end
```

(`serial:string` カラムの追加マイグレーションと unique index は演習 3 で。)

## 📝 今日の研磨(演習)

1. コンソールで `Jewel.new(price: -500)` を作り、`valid?` → `errors.full_messages` →
   `save` → `save!` の順に呼んで、**静かな失敗と例外の違い** を観察してください。
2. **壊す実験:** `validates :stone, inclusion: { in: %w[ruby garnet sapphire quartz] }`
   を追加してから `bin/rails db:seed` を実行してください。もし seed に
   想定外の石があれば `create!` が **例外で止まる** はずです。`create` だったら
   どうなっていたか(=静かにデータが欠ける)も考えてみましょう。
3. `serial:string` カラムを追加するマイグレーションを書き、
   `add_index :jewels, :serial, unique: true` も付けてください。その後コンソールで
   同じ serial のレコードを 2 回 `create!` し、2 回目が
   `ActiveRecord::RecordNotUnique`(DB の砦)で止まることを確認しましょう。

---

一品ものの台帳は堅牢になりました。しかし商売には「お客様」と「注文」が必要で、
それらは互いにつながっています。台帳と台帳を結ぶ——アソシエーションへ。
そして `has_many` の 1 行に、第5章の秘伝が再び火を吹きます。
→ [第11章 台帳をつなぐ](11_associations.md)
