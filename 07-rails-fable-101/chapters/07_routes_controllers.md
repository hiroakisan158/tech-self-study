# 第7章 駅と改札 — ルーティングとコントローラ

## 🚂 今日のお話

鉄道は開通しましたが、駅が「本社前」1 つだけです。お客さんが
「商品一覧を見たい」「この石の詳細を見たい」と思っても降りる駅がありません。

今日は **駅(URL)を設計し、改札係(コントローラ)を配置** します。
そして Rails の駅設計には、世界共通の美しい様式があります——**REST** です。

## routes.rb — 時刻表はここに全部書いてある

`config/routes.rb` は「どの URL に来た列車を、どのコントローラのどのアクションに
案内するか」の一覧表です。まず素朴に書いてみます:

```ruby
Rails.application.routes.draw do
  root "welcome#index"
  get "jewels",     to: "jewels#index"   # 一覧
  get "jewels/:id", to: "jewels#show"    # 詳細(:id は可変部分)
end
```

`get "jewels/:id"` の `:id` が **パスパラメータ** です。`/jewels/1` にも
`/jewels/42` にもマッチし、値はコントローラで `params[:id]` として受け取れます。

## resources — 7 つの駅を 1 行で

しかし Rails 職人は上のようには書きません。CRUD の駅一式は 1 行で建つからです:

```ruby
Rails.application.routes.draw do
  root "welcome#index"
  resources :jewels        # ← これで 7 ルートが生成される
end
```

`bin/rails routes -c jewels` で確認すると:

| HTTP メソッド | パス | アクション | 役割 |
|---|---|---|---|
| GET | /jewels | index | 一覧を見る |
| GET | /jewels/new | new | 新規作成フォームを見る |
| POST | /jewels | create | 作成する |
| GET | /jewels/:id | show | 詳細を見る |
| GET | /jewels/:id/edit | edit | 編集フォームを見る |
| PATCH/PUT | /jewels/:id | update | 更新する |
| DELETE | /jewels/:id | destroy | 削除する |

**「リソース(もの)に対する 7 つの基本操作」** ——これが REST スタイルです。
URL は「もの(名詞)」、操作は HTTP メソッド(動詞)で表す。
`/jewels/delete_item?id=3` のような動詞入り URL は Rails では書きません。

> 🔍 **なぜそうなっているの? — Rails が REST を大衆化した**
> REST は 2000 年の Roy Fielding の論文が起源ですが、長らく学術的な概念でした。
> それを「`resources` と書けば正しい REST 設計になる」という形で
> **一般の開発者の日常に持ち込んだのが Rails 1.2(2007)** です。
> 「URL 設計に悩む時間」を規約で消し去るという、いつもの Rails の手口です。
> 今日、Go で API サーバーを書くときも `GET /users/:id` と設計するのが
> 当たり前になっていますが、その「当たり前」の普及に Rails は大きく貢献しました。
> あなたが Go 教材第14章で書いた HTTP ハンドラの URL 設計は、
> 実は Rails の孫弟子だったのです。

必要な駅だけに絞ることもできます(使わない駅を開けないのが良い設計です):

```ruby
resources :jewels, only: [:index, :show]           # 閲覧専用
resources :orders, except: [:destroy]              # 削除以外
```

## パスヘルパー — URL を文字列で書かない

`resources` はルートと同時に **URL を生成するメソッド(パスヘルパー)** も作ります。

```ruby
jewels_path          # => "/jewels"
jewel_path(1)        # => "/jewels/1"
jewel_path(@jewel)   # モデルを渡せば id を勝手に取り出す
new_jewel_path       # => "/jewels/new"
edit_jewel_path(3)   # => "/jewels/3/edit"
```

ビューでは `link_to`(第8章)と組み合わせます。URL をベタ書きしないことで、
将来パスを変えてもヘルパーの呼び出し側は無傷で済みます。

## コントローラ — 改札係の仕事

```bash
bin/rails generate controller Jewels index show
```

```ruby
# app/controllers/jewels_controller.rb
class JewelsController < ApplicationController
  def index
    @jewels = ["garnet", "ruby", "sapphire"]   # 今は仮データ(第9章で DB に)
  end

  def show
    @name = params[:id]        # /jewels/ruby なら "ruby" が入る
  end
end
```

コントローラの仕事は 3 つだけです:

1. **params からリクエストの情報を取り出す**
2. **モデルに仕事を頼む**(今日は仮データ、第9章から本物)
3. **結果をビューに渡すか、リダイレクトする**

### インスタンス変数がビューに渡る

`@jewels` に注目してください。コントローラで代入した **インスタンス変数(@付き)が、
そのままビューで読めます**。

```erb
<%# app/views/jewels/index.html.erb %>
<h1>商品一覧</h1>
<ul>
  <% @jewels.each do |name| %>
    <li><%= link_to name, jewel_path(name) %></li>
  <% end %>
</ul>
```

第4章で「@変数はそのオブジェクト自身のもの」と学びました。コントローラとビューは
別ファイルなのに、なぜ共有できるのか? ——Rails がビューの描画時に、コントローラの
インスタンス変数を **ビューオブジェクトへコピーしている** からです。規約による
糊付けの一例で、便利さと引き換えに「どの変数がビューに渡るのか宣言がない」という
批判もある部分です(渡す変数は各アクション 1〜2 個に絞るのが作法です)。

## params — 改札で受け取る荷物

`params` はリクエストの情報がすべて入ったハッシュ(風オブジェクト)です。

```ruby
# GET /jewels/3?sort=price&order=desc
params[:id]      # => "3"      パスパラメータ
params[:sort]    # => "price"  クエリパラメータ
params[:order]   # => "desc"
```

> 🐹 **Go との違い①: params は「全部入り・全部文字列」**
> Go では `r.PathValue("id")` と `r.URL.Query().Get("sort")` を **別々に**
> 取り出し、`strconv.Atoi` で明示変換しました。Rails の params はパス・クエリ・
> フォーム由来を **1 つの入れ物に混ぜて** 渡してきます(値はすべて文字列です。
> `params[:id]` は `"3"` であって `3` ではありません!)。
> 便利ですが「どこから来た値か」が曖昧になる欠点があり、その曖昧さが
> かつて大事件を起こしました——第12章の strong parameters で話します。

> 🐍 **Python との違い①: デコレータではなく規約**
> FastAPI では `@app.get("/jewels/{id}")` とデコレータでルートを関数に貼り付け、
> 型ヒントで検証まで行いました。Rails は **ルート定義(routes.rb)と処理
> (コントローラ)を分離** し、両者を「命名規約」で結びます。
> `get "jewels/:id", to: "jewels#show"` の `"jewels#show"` が
> `JewelsController` の `show` メソッドに解決されるのは、文字列からクラス名への
> 規約変換(`jewels` → `JewelsController`)です。第5章で学んだ
> 「文字列からコードを引き当てる」メタプログラミングが、ここでも働いています。

## redirect_to と render — 改札係の 2 つの返事

アクションの最後は「ビューを描く」か「別の駅へ案内する」かのどちらかです。

```ruby
def show
  @jewel = find_jewel(params[:id])
  if @jewel.nil?
    redirect_to jewels_path, alert: "その商品は見つかりませんでした"
    return   # redirect_to は「予約」するだけ。処理は続くので return を忘れずに
  end
  # render は書かなければ show.html.erb が規約で選ばれる
end
```

`redirect_to` がブラウザに「別の URL へ行き直して」と返す(302)のに対し、
`render` はその場で HTML を描いて返します。使い分けは第12章の CRUD 完成編で
体に入れます。

## 💎 完成コード

```ruby
# config/routes.rb
Rails.application.routes.draw do
  root "welcome#index"
  resources :jewels, only: [:index, :show]
end
```

```ruby
# app/controllers/jewels_controller.rb
class JewelsController < ApplicationController
  STONES = {
    "garnet"   => { name: "ガーネット", price: 4800 },
    "ruby"     => { name: "ルビー",     price: 98000 },
    "sapphire" => { name: "サファイア", price: 45000 },
  }.freeze

  def index
    @jewels = STONES
  end

  def show
    @stone = STONES[params[:id]]
    redirect_to jewels_path, alert: "取り扱いのない石です" if @stone.nil?
  end
end
```

```erb
<%# app/views/jewels/index.html.erb %>
<h1>💎 商品一覧</h1>
<ul>
  <% @jewels.each do |slug, stone| %>
    <li><%= link_to "#{stone[:name]} — #{stone[:price]}円", jewel_path(slug) %></li>
  <% end %>
</ul>
```

```erb
<%# app/views/jewels/show.html.erb %>
<h1><%= @stone[:name] %></h1>
<p>価格: <%= @stone[:price] %> 円</p>
<p><%= link_to "← 一覧へ戻る", jewels_path %></p>
```

## 📝 今日の研磨(演習)

1. `bin/rails routes` の出力で、`resources :jewels, only: [:index, :show]` が
   ちょうど 2 行になっていることを確認してください。`only:` を外すと
   7 行に増えることも確認しましょう。
2. **壊す実験:** ブラウザで `/jewels/diamond`(存在しない石)にアクセスし、
   一覧へリダイレクトされることを確認してください。次に `redirect_to` の行を
   コメントアウトすると、show ビューで `nil` に対する `[]` 呼び出しの
   エラーが出ることも見ておきましょう(NoMethodError for nil——Ruby で
   最も出会うエラーです)。
3. `resources :orders, only: [:index]` を routes に足し、対応するコントローラと
   ビューを **generate を使わずに手で** 作ってください(規約——ファイル名と
   クラス名の対応——を体で覚える練習です)。

---

一覧ページはできましたが、`<li>` を手書きするショーウィンドウは寂しすぎます。
レイアウト、部分テンプレート、ヘルパー——**見せ方の道具一式** を学びます。
→ [第8章 ショーウィンドウ](08_views.md)
