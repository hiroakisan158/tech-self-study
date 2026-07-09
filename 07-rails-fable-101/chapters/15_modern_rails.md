# 第15章 動く売り場 — Hotwire・Active Job・API モード

## 🚂 今日のお話

お客様から要望が届きました。「絞り込みのたびにページ全体が白くなるのが気になる」
「注文完了メールが届くまで画面が固まる」。

現代の Web らしい滑らかさ——しかし親方は言います。「JavaScript の大工事は
やらん。**Rails には Rails のやり方がある**」。今日は現代 Rails の 3 本柱、
Hotwire・Active Job・API モードを一望します。

## Hotwire — 「HTML を配信する」という反撃

React/Next.js の世界(Web 3 部作)では、サーバーは JSON を返し、
ブラウザ側の JS が DOM を組み立てました。Hotwire は正反対の賭けです:

**「サーバーが HTML を返し続ける。ただし差し替えを賢くやれば、体験は SPA と
変わらない」**

Hotwire は Rails 7 から標準装備で、3 つの部品からなります。

### Turbo Drive — 何もしなくても SPA 風(すでに動いている)

実は、あなたのアプリのリンクはすでに **ページ全体を再読み込みしていません**。
Turbo Drive がリンククリックを横取りし、fetch で HTML を取得して
`<body>` だけ差し替えています。`<head>` の CSS/JS を再評価しないので速い。
**設定ゼロ、コード変更ゼロ** で、全 Rails アプリが最初からこの挙動です。

第12章で `status: :see_other` や `:unprocessable_entity` を几帳面に
書いたのは、Turbo がリダイレクトと再描画を区別するのに必要だったからです。
伏線回収です。

### Turbo Frames — ページの一部だけを差し替える

「絞り込みで商品グリッドだけ更新したい」なら、その領域を枠(フレーム)で囲みます:

```erb
<%# index.html.erb %>
<%= form_with url: jewels_path, method: :get,
      data: { turbo_frame: "jewel_list" } do |f| %>
  <%= f.select :stone, options_for_select(Jewel.distinct.pluck(:stone)),
        include_blank: "すべての石" %>
  <%= f.submit "絞り込む" %>
<% end %>

<%= turbo_frame_tag "jewel_list" do %>
  <%= render partial: "jewel", collection: @jewels %>
<% end %>
```

フォーム送信への応答 HTML から、Turbo が **同じ id のフレームだけ** を抜き出して
差し替えます。コントローラは今までどおり index を描くだけ。
**JavaScript を 1 行も書いていない** ことを確認してください。

### Turbo Streams — 複数箇所の更新・リアルタイム配信

「明細を追加したら、一覧にも合計金額にもカート個数バッジにも反映したい」——
複数箇所の更新は Turbo Streams で「変更の指示書」を返します:

```erb
<%# app/views/line_items/create.turbo_stream.erb %>
<%= turbo_stream.append "cart_items", partial: "line_items/line_item",
      locals: { line_item: @line_item } %>
<%= turbo_stream.update "cart_total", @order.total_price %>
```

さらに Action Cable(WebSocket)と組むと、**他のユーザーの画面にも**
同じ指示書を配信できます(在庫が減ったら全員の画面で「売切」になる、など)。

### Stimulus — それでも JS が要る場面の「控えめな」流儀

ドロップダウンの開閉のような純クライアント側の動きには Stimulus を使います:

```html
<div data-controller="dropdown">
  <button data-action="click->dropdown#toggle">メニュー</button>
  <div data-dropdown-target="menu" hidden>...</div>
</div>
```

HTML の data 属性が JS の小さなコントローラを呼ぶ、という設計です。
React のように「UI 全体を JS で所有する」のではなく、
**サーバー製 HTML に、振る舞いをちょい足しする**——役割が逆転しています。

> 🔍 **なぜそうなっているの? — One Person Framework という宣言**
> 2021 年、DHH は Rails 7 を「**The One Person Framework**」と宣言しました。
> 「フロント専門家とバックエンド専門家のチームがなければ現代的 Web アプリが
> 作れない、という状況への反逆。**たった 1 人でフルスタックの
> アプリを作り、運用できる道具** であり続ける」。
> SPA 全盛期に Rails が下した「JSON API 化して React に道を譲るのではなく、
> HTML 配信を進化させる」という決断は、当時は懐疑的に見られましたが、
> Next.js が Server Components で「サーバーで描く」方向へ揺り戻した今、
> 先見的だったと再評価されています。37signals(DHH の会社)が
> 数人のチームで HEY や Basecamp を運用し続けていること自体が、
> この思想の実証実験です。

## Active Job — 裏で走る仕事

注文完了メールの送信に 3 秒かかるとして、その 3 秒お客様を待たせるのは
悪手です。「後でやる仕事」はジョブキューに逃がします:

```bash
bin/rails generate job OrderConfirmation
```

```ruby
# app/jobs/order_confirmation_job.rb
class OrderConfirmationJob < ApplicationJob
  queue_as :default

  def perform(order)
    OrderMailer.confirmation(order).deliver_now   # メール送信(重い処理の代表)
  end
end
```

```ruby
# コントローラからは「予約」するだけ(一瞬で戻る)
OrderConfirmationJob.perform_later(@order)
```

Rails 8 では、ジョブの実行基盤として **Solid Queue** が標準になりました。
従来は Redis + Sidekiq という追加インフラが定番でしたが、Solid Queue は
**ジョブを普通の DB テーブルに積む** ので、追加ミドルウェアなしで動きます。
同様に、キャッシュは Solid Cache、WebSocket は Solid Cable——
「Redis がなくても本番運用できる」のが Rails 8 の合言葉です
(これも One Person Framework の延長線です)。

> 🐹 **Go との違い①: goroutine では代わりにならない**
> Go なら `go sendMail(order)` で済むのでは?と思うかもしれません。
> 実は Go でも本番では済みません——プロセスが再起動したら実行中の
> goroutine は消えますし、失敗時のリトライも自前です。
> Active Job(と Solid Queue)は **永続化・リトライ・スケジュール実行・
> 失敗の記録** まで面倒を見る仕組みで、Go の世界でも同種の道具
> (River、Asynq など)を結局導入します。「軽量スレッド」と
> 「永続ジョブキュー」は別のレイヤーの道具です。

## API モード — Rails を JSON 工場にする

「フロントは Next.js で作りたい(または既にある)」場合、Rails は
ビュー層を持たない **API モード** で動かせます:

```bash
rails new kogyokudo-api --api
```

コントローラは JSON を返す係になります:

```ruby
class Api::JewelsController < ApplicationController
  def index
    render json: Jewel.in_stock.order(:price)
  end

  def show
    render json: Jewel.find(params[:id])
  end
end
```

モデル・マイグレーション・バリデーション・アソシエーション・ジョブ・テスト——
この教材で学んだことの **8 割はそのまま使えます**。消えるのは第8章(ビュー)と
本章の Hotwire だけです。

> 🌐 **Web 3 部作との接続点**
> [Next.js 教材](../../06-nextjs-fable-101/README.md)を学んだ人は、ここで
> 2 つの世界が接続します。実務でよくある構成は
> 「**Rails API(この教材)+ Next.js フロント(Web 3 部作)**」——
> Rails の得意な DB・業務ロジック・管理画面と、React の得意な
> リッチな UI を分業させる形です。一方、社内向けツールや小規模サービスなら
> 「Hotwire で全部 Rails」が圧倒的に速い。**どちらも選べるのが、
> 両方の教材を修めたあなたの武器です。**

## 💎 完成コード

本章は「一望」の章なので、手を動かす最小セットはこれだけです:

1. index の絞り込みフォームを Turbo Frame 化(上記コード)
2. `OrderConfirmationJob` を作り、コンソールから
   `OrderConfirmationJob.perform_later(Order.first)` →
   `bin/jobs`(Solid Queue のワーカー起動)で実行を確認

## 📝 今日の研磨(演習)

1. ブラウザの開発者ツール(ネットワークタブ)を開いて商品一覧のリンクを
   クリックし、**ページ遷移が fetch リクエストになっている**(Turbo Drive)ことを
   確認してください。`<body>` だけが差し替わる様子も Elements タブで見えます。
2. 絞り込みフォームを Turbo Frame 化する前後で、ネットワークタブの挙動を
   比べてください。フレーム化後は応答 HTML が同じでも **描画される範囲** が
   変わります。
3. **壊す実験:** `turbo_frame_tag "jewel_list"` の id をフォーム側とわざと
   食い違わせて送信し、「Content missing」表示を観察してください。
   フレームの対応が **id の一致だけ** で決まっていることが分かります。

---

道具はすべて揃いました。最終章では紅玉堂オンラインを完成させます——
ログイン機能、本番環境への出荷(デプロイ)、そして工房を巣立つあなたへの
「卒業後の地図」です。
→ [第16章 卒業制作](16_final.md)
