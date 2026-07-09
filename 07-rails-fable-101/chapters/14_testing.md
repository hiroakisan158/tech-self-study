# 第14章 出荷前検査 — Rails のテスト

## 🚂 今日のお話

先週、価格計算の修正が「たまたま」注文合計を壊し、お客様への請求額が
狂いかけました。親方は静かに言いました。「出荷前検査のない工房に、未来はない」

Ruby は動的型付けの言語です。コンパイラは typo も型違いも教えてくれません。
**Ruby 職人の品質保証はテストがすべて** ——だからこそ Ruby コミュニティは
世界のテスト文化を牽引してきました。今日はその伝統に入門します。

## Rails のテストは「最初から」ある

`rails new` した瞬間から `test/` ディレクトリと minitest が付いてきます。
第9章の `generate model` は、実はテストの雛形も一緒に作っていました:

```
test/
├── models/jewel_test.rb        ← モデルの検査
├── controllers/                ← リクエスト単位の検査
├── system/                     ← ブラウザを動かす検査
├── fixtures/jewels.yml         ← テスト用データ
└── test_helper.rb
```

```bash
bin/rails test          # 全テスト実行
bin/rails test test/models/jewel_test.rb   # 1 ファイルだけ
```

テスト用の DB は開発用と別に自動で用意され、**各テストはトランザクションで
包まれて、終わると自動ロールバック** されます。テスト同士がデータを
汚し合わない仕掛けが最初から入っています。

## モデルテスト — 検品ルールを検査する

```ruby
# test/models/jewel_test.rb
require "test_helper"

class JewelTest < ActiveSupport::TestCase
  test "名前と正の価格があれば有効" do
    jewel = Jewel.new(name: "紅玉の指輪", stone: "ruby", price: 98_000, stock: 1)
    assert jewel.valid?
  end

  test "名前が空なら無効" do
    jewel = Jewel.new(name: "", stone: "ruby", price: 1000)
    assert_not jewel.valid?
    assert_includes jewel.errors[:name], "can't be blank"
  end

  test "価格が負なら無効" do
    jewel = Jewel.new(name: "怪しい石", stone: "quartz", price: -500)
    assert_not jewel.valid?
  end
end
```

`test "..." do ... end` は **ブロックを取るクラスマクロ** です(built on 第3+5章)。
テスト名を日本語の文章で書けるので、失敗時の出力がそのまま検査報告書になります。

> 🐹 **Go との違い①: テーブル駆動 vs 1 テスト 1 ケース**
> Go では `[]struct{...}` にケースを並べて for で回すテーブル駆動が正道でした。
> Ruby でも似たことはできますが、文化は「**1 つの振る舞いに 1 つの test ブロック、
> 名前は仕様を文章で**」です。テストの一覧がそのまま仕様書として読めることを
> 重視します。Go のテストが「網羅の道具」なら、Ruby のテストは
> 「仕様の記述 + 型検査の代役」——動的型の言語では、テストが
> コンパイラの仕事(typo 検出、リグレッション検出)まで担うからです。

## fixtures — 検査用の見本品

`test/fixtures/jewels.yml` に書いた見本データは、全テストから名前で呼べます:

```yaml
# test/fixtures/jewels.yml
ruby_ring:
  name: 紅玉の指輪
  stone: ruby
  price: 98000
  stock: 2

cheap_quartz:
  name: 水晶の風鈴
  stone: quartz
  price: 1200
  stock: 0
```

```ruby
test "在庫のある石だけが in_stock に含まれる" do
  assert_includes Jewel.in_stock, jewels(:ruby_ring)
  assert_not_includes Jewel.in_stock, jewels(:cheap_quartz)
end
```

`jewels(:ruby_ring)` で fixture がモデルとして手に入ります。
(大規模になると fixture の管理が辛くなり、factory_bot という gem で
「その場で組み立てる」方式に移行するチームが多数派です——実務で出会ったら
「fixture の代替品」と思えば OK です。)

## コントローラ(リクエスト)テスト — 改札の検査

```ruby
# test/controllers/jewels_controller_test.rb
require "test_helper"

class JewelsControllerTest < ActionDispatch::IntegrationTest
  test "一覧が表示できる" do
    get jewels_url
    assert_response :success
  end

  test "商品を登録できる" do
    assert_difference("Jewel.count", 1) do
      post jewels_url, params: { jewel: { name: "新しい石", stone: "garnet",
                                          price: 3000, stock: 5 } }
    end
    assert_redirected_to jewel_url(Jewel.last)
  end

  test "検品落ちなら 422 でフォーム再表示" do
    post jewels_url, params: { jewel: { name: "", price: -1 } }
    assert_response :unprocessable_entity
  end
end
```

`assert_difference("Jewel.count", 1)` は「このブロックの前後でレコードが
1 件増えること」の検査です。HTTP を実際に通す(ルーティング → コントローラ →
モデル → ビューまで貫通する)ので、第12章で書いた CRUD の型全体を守れます。

## システムテスト — ブラウザごと動かす

```ruby
# test/system/jewel_purchases_test.rb
require "application_system_test_case"

class JewelPurchasesTest < ApplicationSystemTestCase
  test "商品を登録して一覧に表示される" do
    visit new_jewel_url
    fill_in "商品名", with: "翡翠の帯留め"
    fill_in "価格",   with: 32000
    click_on "Create Jewel"

    assert_text "商品を登録しました"
    assert_text "翡翠の帯留め"
  end
end
```

Capybara というライブラリが本物のブラウザ(ヘッドレス Chrome)を操作します。
`visit` `fill_in` `click_on` ——**人間の操作がそのまま動詞になっている** ことに
注目してください。これも Ruby の DSL 芸です。実行は `bin/rails test:system`。

遅い代わりに「ユーザーに見える壊れ方」を検出できるのはシステムテストだけです。
配分の定石は **モデルテスト多め・リクエストテスト中くらい・システムテスト少数精鋭**
(いわゆるテストピラミッド)です。

## RSpec — 実務でおそらく出会うもう 1 つの流儀

ここまで使った minitest は Rails 標準ですが、実務の Rails プロジェクトでは
**RSpec** という別のテストフレームワークのシェアが高いです。見た目が大きく違います:

```ruby
# RSpec の書き方(gem 'rspec-rails' を導入した場合)
RSpec.describe Jewel, type: :model do
  describe "#valid?" do
    context "価格が負のとき" do
      it "無効になる" do
        jewel = Jewel.new(name: "怪しい石", price: -500)
        expect(jewel).not_to be_valid
      end
    end
  end
end
```

`describe` / `context` / `it` / `expect(...).to` ——テストを
**振る舞いの仕様書(スペック)として英語の文章のように書く** 思想
(BDD: Behavior-Driven Development)から生まれた DSL です。
minitest が「素の Ruby で書く質実剛健派」、RSpec が「DSL で表現力を極める派」。
どちらも中身は第5章の技術(クラスマクロ + ブロック + method_missing)でできています。
**minitest で概念を掴んでおけば、RSpec は書き方の翻訳だけで移行できます。**

> 🔍 **なぜそうなっているの? — テスト文化の総本山としての Ruby**
> 「動的型だからテストが必須」という必要に迫られた事情だけでなく、
> Ruby コミュニティはテストを **文化として** 育てました。2000 年代後半、
> RSpec や Cucumber(自然言語でテストを書く試み)は BDD ムーブメントの
> 中心にいましたし、「テストのないコードはレガシーコード」という規範を
> 業界に広めたのも Rails 界隈の声が大きかった。DHH が Rails に
> テストディレクトリを標準装備した(=テストを書かない構成を
> そもそも用意しなかった)ことの影響は計り知れません。
> 皮肉なことに、型のない言語が **一番テストに真剣な文化** を生んだのです。
> Go の `go test` 標準装備(教材第15章)も、この流れの後継者です。

## 📝 今日の研磨(演習)

1. 第10章で作った `normalizes :name`(前後の空白除去)のテストを書いてください。
   「`"  紅玉  "` で作ると name が `"紅玉"` になる」を assert します。
2. 第11章の `Order#total_price` のテストを書いてください。fixtures に
   orders / line_items を用意する必要があります(customers も)。
   fixture 間の関連は `customer: rei` のように **fixture 名で** 参照できます。
3. **壊す実験(テストの存在意義):** `total_price` の実装をわざと
   `* item.quantity` 抜きに変え、`bin/rails test` が **赤くなる** ことを
   確認してから戻してください。「壊したら気づける」状態こそが、
   この章で手に入れた資産です。

---

検査体制が整い、安心してコードを変えられるようになりました。
仕上げに、売り場を現代的にします——ページ遷移なしで動く UI、裏で走る仕事、
そして「Rails は今どこへ向かっているのか」。
→ [第15章 動く売り場](15_modern_rails.md)
