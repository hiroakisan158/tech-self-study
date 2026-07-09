# 第12章 注文受付 — フォームと CRUD の完成

## 🚂 今日のお話

いよいよ Web から商品を登録・編集できるようにします。管理画面の第一歩です。
今日で **CRUD の 7 アクションがすべて埋まり**、Rails アプリの「基本形」が完成します。

この基本形は Rails 職人が千回書いてきた **型** です。一度体に入れば、
世のほとんどの Rails コントローラが読めるようになります。

## form_with — モデルから生えるフォーム

```erb
<%# app/views/jewels/new.html.erb %>
<h1>新しい商品を登録</h1>
<%= render "form", jewel: @jewel %>
```

```erb
<%# app/views/jewels/_form.html.erb — new と edit で共用するパーシャル %>
<%= form_with model: jewel do |f| %>
  <% if jewel.errors.any? %>
    <div class="errors">
      <h2>検品で <%= jewel.errors.count %> 件の問題が見つかりました:</h2>
      <ul>
        <% jewel.errors.full_messages.each do |msg| %>
          <li><%= msg %></li>
        <% end %>
      </ul>
    </div>
  <% end %>

  <div><%= f.label :name, "商品名" %>  <%= f.text_field :name %></div>
  <div><%= f.label :stone, "石の種類" %> <%= f.text_field :stone %></div>
  <div><%= f.label :price, "価格" %>   <%= f.number_field :price %></div>
  <div><%= f.label :stock, "在庫数" %>  <%= f.number_field :stock %></div>

  <%= f.submit %>
<% end %>
```

`form_with model: jewel` の賢さは、**モデルの状態を見て挙動を変える** ところです:

- `jewel` が新規(未保存)→ `POST /jewels` に送信、ボタンは「Create Jewel」
- `jewel` が保存済み → `PATCH /jewels/:id` に送信、ボタンは「Update Jewel」
- 各フィールドには **モデルの現在値が自動で入る**(編集画面がタダで手に入る)

だからフォームは 1 枚のパーシャルで new / edit 兼用にできます。
CSRF トークン(なりすまし送信の防止)も自動で埋め込まれています。

## コントローラの完成形 — 千回書かれた型

```ruby
# app/controllers/jewels_controller.rb
class JewelsController < ApplicationController
  before_action :set_jewel, only: [:show, :edit, :update, :destroy]

  def index
    @jewels = Jewel.order(:price)
  end

  def show
  end

  def new
    @jewel = Jewel.new
  end

  def create
    @jewel = Jewel.new(jewel_params)
    if @jewel.save
      redirect_to @jewel, notice: "商品を登録しました"
    else
      render :new, status: :unprocessable_entity
    end
  end

  def edit
  end

  def update
    if @jewel.update(jewel_params)
      redirect_to @jewel, notice: "商品を更新しました"
    else
      render :edit, status: :unprocessable_entity
    end
  end

  def destroy
    @jewel.destroy
    redirect_to jewels_path, notice: "商品を削除しました", status: :see_other
  end

  private

  def set_jewel
    @jewel = Jewel.find(params[:id])
  end

  def jewel_params
    params.expect(jewel: [:name, :stone, :price, :stock])
  end
end
```

読みどころを順番に:

- **before_action** — 4 アクション共通の「対象を探す」処理を 1 箇所に。
  これもクラスマクロです(もう驚きませんね)。
- **create / update の分岐** — 成功なら `redirect_to`(二重送信防止のため、
  更新系の成功後は必ずリダイレクト)。失敗なら `render` で **同じリクエスト内で**
  フォームを再表示。`@jewel` が errors を抱えたままビューに渡るので、
  パーシャルのエラー表示が動きます。
- **redirect_to @jewel** — モデルを渡すと `jewel_path(@jewel)` に解決されます。
- **status: :unprocessable_entity(422)/ :see_other(303)** — 第15章の
  Turbo が正しく動くために必要な HTTP ステータスです。今は「型の一部」として
  そのまま書いてください。

## strong parameters — 10 数年前の大事件が生んだ関所

`jewel_params` が今日の最重要ポイントです。なぜ `params[:jewel]` を
そのまま `Jewel.new` に渡してはいけないのか。

```ruby
# ❌ 絶対にやってはいけない
@jewel = Jewel.new(params[:jewel])
```

フォームから来る `params` は、**攻撃者が自由に細工できます**。フォームに
ない項目でも、リクエストに `jewel[admin]=true` を混ぜて送るのは一瞬です。
モデルに `admin` カラムがあれば、そのまま書き込まれてしまいます
(マスアサインメント攻撃)。

そこで **「受け取ってよい項目を明示的に列挙する」** 関所を通します:

```ruby
def jewel_params
  params.expect(jewel: [:name, :stone, :price, :stock])   # Rails 8 の書き方
  # Rails 7 以前のコードでは params.require(:jewel).permit(:name, ...) を見かけます
end
```

列挙にない項目は黙って捨てられます。ホワイトリスト方式です。

> 🔍 **なぜそうなっているの? — GitHub が破られた 2012 年 3 月 4 日**
> かつての Rails はモデル側の `attr_accessible` で守る設計でしたが、
> 「書き忘れたら全カラム書き込み放題」という危うい既定値でした。
> 2012 年、ロシア人開発者 Egor Homakov がこの穴を **GitHub 本体で実証** します。
> 公開鍵の紐付け先ユーザー ID をリクエストに細工して混ぜ、
> **Rails 公式リポジトリに自分の公開鍵を登録し、コミット権限を取得** したのです
> (悪用はせず、警告のためのデモでした)。この事件で議論は決着し、
> 「守りはモデルではなくコントローラで、デフォルト拒否で」という
> strong parameters が Rails 4 で標準になりました。
> あなたが書く `jewel_params` の 1 行は、この事件の教訓の結晶です。

> 🐹 **Go との違い①: 構造体が関所になる世界との対比**
> Go では `json.Unmarshal(body, &req)` の `req` を **専用の構造体** として定義し、
> そこに無いフィールドは最初から受け取りようがありませんでした。
> 型がホワイトリストの役割を果たしていたのです。Rails の params は
> 型のない自由なハッシュなので、**関所を手続きで作る** 必要がありました。
> 静的型の言語では設計が守ってくれるものを、動的型の言語では規約と道具で守る——
> この対比は Web 開発のあらゆる場面で顔を出します。

## ルートとリンクの仕上げ

```ruby
# config/routes.rb
resources :jewels    # only: を外して 7 アクション全開放
```

```erb
<%# index.html.erb に追記 %>
<%= link_to "＋ 新しい商品を登録", new_jewel_path %>

<%# show.html.erb に追記 %>
<%= link_to "編集", edit_jewel_path(@jewel) %>
<%= button_to "削除", jewel_path(@jewel), method: :delete,
      data: { turbo_confirm: "本当に削除しますか?" } %>
```

削除だけ `button_to` なのは、GET リンクでデータを消してはいけないからです
(クローラーが踏んだだけで全商品が消える事故は、Web 黎明期の実話です)。

## scaffold — 今日書いた全部を 1 コマンドで

実は、今日手で書いたものほぼすべてを Rails は 1 コマンドで生成できます:

```bash
bin/rails generate scaffold Material name:string price:integer
```

モデル・マイグレーション・7 アクションのコントローラ・全ビュー・ルートが
一度に生まれます。「15 分でブログ」の正体はこれです。

では、なぜ scaffold を先に教えなかったのか。**生成されたコードは、
自分で書けるものの時短としてしか役に立たない** からです。読めないコードを
量産する魔法の杖として使うと、カスタマイズが必要になった瞬間に詰みます。
今日、型を手で書いたあなたは scaffold を「答え合わせ」として使えます。

## 📝 今日の研磨(演習)

1. ブラウザから商品の登録 → 編集 → 削除を一巡してください。わざと価格を
   `-100` にして、**検品エラーがフォームに表示され、入力値が残っている** ことを
   確認しましょう(render のおかげです。redirect だったら入力は消えます)。
2. **壊す実験(マスアサインメント編):** `jewel_params` の列挙に無い
   `:serial` をフォームの hidden フィールドとして手で追加し
   (ブラウザの開発者ツールで `<input name="jewel[serial]" value="HACK">` を注入)、
   送信しても serial が変わらないことをコンソールで確認してください。
   関所が仕事をした瞬間です。
3. `bin/rails generate scaffold Material name:string price:integer` を実行し、
   生成されたコントローラと今日の手書き版を **見比べて** ください。
   確認後は `bin/rails destroy scaffold Material` で片付けられます
   (generate の逆コマンドです)。

---

店は開きました。しかし商品が増えるにつれ、一覧が遅くなってきました。
台帳係が **同じ棚に何百回も往復している**(N+1 問題)のです。
検索と絞り込みの正しい道具——クエリの章へ。
→ [第13章 在庫検索](13_queries.md)
