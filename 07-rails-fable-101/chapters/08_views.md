# 第8章 ショーウィンドウ — ERB・レイアウト・パーシャル

## 🚂 今日のお話

「商品は並んだが、これでは露店だ」と親方。「紅玉堂ののれん、統一された陳列棚、
値札の書式——**店構え** を整えるぞ」

今日はビューの道具一式です。ERB テンプレート、全ページ共通のレイアウト、
使い回せる陳列棚(パーシャル)、そして値札職人(ヘルパー)。

## ERB — テンプレートではなく「HTML に埋め込まれた Ruby」

ERB(Embedded Ruby)のタグは 2 種類だけです:

```erb
<%= 式 %>   出力する(= あり)
<%  文  %>   実行するだけで出力しない(= なし)
```

```erb
<% if @jewels.any? %>
  <p>本日は <%= @jewels.size %> 点ございます</p>
<% else %>
  <p>ただいま品切れ中です</p>
<% end %>
```

> 🐍 **Python との違い①: Jinja2 は独自言語、ERB は Ruby そのもの**
> Jinja2 の `{% for item in items %}` はテンプレート専用のミニ言語で、
> Python とは別の文法・別のフィルタ体系を覚える必要がありました。
> ERB のタグの中身は **100% ただの Ruby** です。`each` もメソッドチェーンも
> 三項演算子も、第1〜5章で学んだものがそのまま動きます。新しく覚えることが
> 少ない代わりに、「テンプレートに何でも書けてしまう」自由もあります——
> ビューにビジネスロジックを書き始めたら手を止めるのが職人の規律です。

> 🐹 **Go との違い①: html/template の不自由は安全のため、では ERB は?**
> Go の `html/template` は意図的に機能を絞り、ロジックをテンプレートに
> 書かせない設計でした。ERB は全開放ですが、**XSS 対策のレベルは同等** です。
> Rails の `<%= %>` は出力を **デフォルトで HTML エスケープ** します。
> `<%= "<script>alert(1)</script>" %>` は無害な文字列として表示されます。
> エスケープを外す `raw` や `html_safe` は、ユーザー入力に対しては
> **絶対に使ってはいけません**(使ってよいのは自分で組み立てた安全な HTML だけ)。

## レイアウト — 全ページ共通の「店構え」

すべてのページに共通するヘッダー・フッター・`<head>` は、
`app/views/layouts/application.html.erb` に 1 回だけ書きます。

```erb
<%# app/views/layouts/application.html.erb(抜粋) %>
<!DOCTYPE html>
<html>
  <head>
    <title>紅玉堂オンライン</title>
    <%= csrf_meta_tags %>
    <%= stylesheet_link_tag "application" %>
  </head>
  <body>
    <header>
      <h1><%= link_to "💎 紅玉堂", root_path %></h1>
      <nav><%= link_to "商品一覧", jewels_path %></nav>
    </header>

    <% if notice %><p class="notice"><%= notice %></p><% end %>
    <% if alert  %><p class="alert"><%= alert %></p><% end %>

    <main>
      <%= yield %>   <%# ← 各アクションのビューがここに差し込まれる %>
    </main>

    <footer>&copy; 紅玉堂 — 山あいの町から鉄道に乗せて</footer>
  </body>
</html>
```

**`<%= yield %>`** に注目してください。第3章で学んだ `yield` と同じ発想です——
「レイアウトという枠を実行し、中身(各ページのビュー)を差し込む」。
`index.html.erb` や `show.html.erb` には `<html>` や `<head>` を書きません。
中身だけを書けば、自動でレイアウトに包まれます。

`notice` と `alert` は **flash メッセージ**(「保存しました」のような 1 回きりの
通知)の定番 2 種です。前章の `redirect_to jewels_path, alert: "..."` が
ここに表示されていたのです。

## パーシャル — 使い回せる陳列棚

商品カードは一覧でも検索結果でもトップページでも使います。
部分テンプレート(パーシャル)に切り出しましょう。**ファイル名の先頭に `_`** を付けます。

```erb
<%# app/views/jewels/_jewel.html.erb — 商品カード1枚分 %>
<div class="jewel-card">
  <h2><%= link_to jewel[:name], jewel_path(jewel[:slug]) %></h2>
  <p class="price"><%= number_to_currency(jewel[:price], unit: "¥", precision: 0) %></p>
</div>
```

使う側は `render`:

```erb
<%# 1枚だけ描く %>
<%= render "jewel", jewel: @featured %>

<%# コレクションをまとめて描く(each 不要!) %>
<%= render partial: "jewel", collection: @jewels, as: :jewel %>
```

パーシャル内では、渡した **ローカル変数**(`jewel`)を使います。
コントローラの `@` 変数をパーシャルの中で直接読むこともできてしまいますが、
**パーシャルは引数(locals)だけに依存させる** のが再利用性を守る作法です
(関数がグローバル変数を読まないのと同じ理屈です)。

## ヘルパー — 値札職人たち

Rails には HTML 生成の便利メソッド(ヘルパー)が大量に用意されています。

```erb
<%= link_to "商品一覧", jewels_path %>
<%= link_to "削除", jewel_path(@jewel), data: { turbo_method: :delete,
      turbo_confirm: "本当に削除しますか?" } %>
<%= image_tag "ruby.jpg", alt: "ルビーの写真", width: 300 %>
<%= number_to_currency(98000, unit: "¥", precision: 0) %>   <%# => ¥98,000 %>
<%= number_with_delimiter(1234567) %>                        <%# => 1,234,567 %>
<%= truncate("長い商品説明文……", length: 30) %>
<%= time_ago_in_words(3.days.ago) %>                         <%# => 3日 %>
```

自作もできます。`app/helpers/` のモジュールに書けば、全ビューから呼べます:

```ruby
# app/helpers/jewels_helper.rb
module JewelsHelper
  def stock_badge(stock)
    if stock.zero?
      tag.span "在庫切れ", class: "badge sold-out"
    elsif stock < 3
      tag.span "残りわずか", class: "badge few"
    else
      tag.span "在庫あり", class: "badge in-stock"
    end
  end
end
```

```erb
<%= stock_badge(jewel[:stock]) %>
```

「ビューに if が 3 つ以上並んだらヘルパーに切り出す」が目安です。
第4章の mixin を思い出してください——ヘルパーモジュールがビューに
include される、あの仕組みがここでも動いています。

> 🔍 **なぜそうなっているの? — サーバーで HTML を組み立てるという選択**
> React(Web 3 部作)を知っている人は「え、サーバーで HTML を文字列連結するの?
> 古くない?」と思うかもしれません。実際、2015〜2020 年頃は
> 「Rails は API だけ、画面は React/SPA」という構成が主流になりかけました。
> しかし Rails 陣営は **Hotwire**(第15章)で反撃します——「HTML をサーバーで
> 描いて差分だけ送れば、JS をほぼ書かずに SPA 並みの体験が作れる」。
> ERB + パーシャルという「古い」道具は、Hotwire の部品として現役復帰しました。
> 「サーバー主導 HTML」対「クライアント主導 JS」の振り子は今も揺れていて、
> Next.js の Server Components はむしろ **Rails 側への揺り戻し** です。
> どちらの世界も知っているあなたは、この対立の良い観客席にいます。

## 💎 完成コード

上記のレイアウト・パーシャル・ヘルパーを配置し、一覧ページを組み直します:

```erb
<%# app/views/jewels/index.html.erb %>
<h1>💎 商品一覧</h1>

<% if @jewels.any? %>
  <div class="jewel-grid">
    <% @jewels.each do |slug, stone| %>
      <%= render "jewel", jewel: stone.merge(slug: slug) %>
    <% end %>
  </div>
<% else %>
  <p>ただいま品切れ中です。次の入荷をお待ちください。</p>
<% end %>
```

```css
/* app/assets/stylesheets/application.css に追記 */
.jewel-card { border: 1px solid #ccc; padding: 1rem; margin: .5rem; }
.price { color: #a01a2f; font-weight: bold; }
.badge.sold-out { color: white; background: #888; padding: 2px 8px; }
```

## 📝 今日の研磨(演習)

1. フッターに「営業時間のご案内」を追加し、**全ページに反映される** ことを
   確認してください(レイアウトを 1 箇所直せば全部変わる、が体感できます)。
2. **壊す実験(XSS 編):** show ビューに `<%= "<b>太字?</b>" %>` と書いて、
   タグがエスケープされて **文字として表示される** ことを確認してください。
   次に `<%= raw "<b>太字?</b>" %>` にすると本当に太字になります。
   この違いが XSS 防御の最前線です。確認したら raw は消しましょう。
3. `stock_badge` ヘルパーを実装し、STONES の各石に `stock:` を足して
   一覧にバッジを表示してください。「残りわずか」の石を作って動作確認を。

---

店構えは整いました。しかし商品データはまだコントローラにハードコードされた
ハッシュです。台帳を **本物のデータベース** に移す時が来ました。
Rails の心臓部、ActiveRecord の登場です。
→ [第9章 商品台帳](09_activerecord_basics.md)
