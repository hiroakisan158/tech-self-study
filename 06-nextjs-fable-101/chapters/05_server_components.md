# 第5章 厨房の革命 — Server Components

## 🍽️ 今日のお話

今日は Bistro Next の心臓部、厨房の扉を開けます。

ここまで書いてきたページには、実は驚くべき事実が隠れています。第 4 章の
`MenuItemPage` は `async` 関数でした。[React 教材](../../05-react-fable-101/README.md)では
一度も出てこなかった書き方です。コンポーネントの中で `await` ができるなら、
[第 14 章であれほど苦労した fetch の 3 局面管理](../../05-react-fable-101/chapters/14_data_fetching.md)は
どこへ行ったのでしょう?

答え: **このコンポーネントは、ブラウザではなくサーバーで実行されているからです。**
App Router のコンポーネントは、**既定で全部 Server Component** です。あなたは
第 1 章からずっと、知らないうちに厨房で調理していたのです。

## Server Component とは何か — 「厨房でだけ実行される React」

定義は一行です:

> **Server Component = サーバー(厨房)で実行され、結果だけがブラウザ(客席)に
> 届く React コンポーネント。その JS コードは客席には一切送られない。**

[React 第 10 章](../../05-react-fable-101/chapters/10_rendering.md)で「レンダリングとは、
コンポーネント関数を呼んで設計図オブジェクトを得ること」と学びました。そして
「**稽古場をサーバーに置く**方向へ進化している」と予告しました。それがこれです。
関数を呼ぶ場所が Node.js に変わっただけで、**React のレンダリングモデル自体は
何も変わっていません**。

厨房で実行されることの意味を、実際に確かめましょう:

```tsx
// app/menu/page.tsx — これは厨房で動いている
import { menuItems } from "../../data/menu";

export default function MenuPage() {
  console.log("🍳 お品書きを調理中…");   // ← どこに表示される?
  return ( /* 省略 */ );
}
```

`/menu` をブラウザで開くと、このログは **ブラウザのコンソールには出ません**。
**`npm run dev` を実行しているターミナルに出ます。** そこがこのコードの実行場所——
あなたの Node.js プロセスです。

## 厨房でできること — ブラウザの制約からの解放

サーバーで動くということは、[TS 教材後半で使った Node.js の全能力](../../04-typescript-fable-101/chapters/14_runtime_validation.md)が
コンポーネントの中で使えるということです:

```tsx
// app/reviews/page.tsx — お客さまの声(ファイルから直接読む!)
import { readFile } from "node:fs/promises";
import { z } from "zod";

const ReviewsSchema = z.array(
  z.object({ id: z.number(), author: z.string(), stars: z.number(), comment: z.string() })
);

export default async function ReviewsPage() {
  // ① コンポーネントの中で await できる(async コンポーネントはサーバー専用の特権)
  const raw = await readFile("data/reviews.json", "utf-8");

  // ② 城壁の外のデータには門番を(TS 第 14 章の作法そのまま)
  const reviews = ReviewsSchema.parse(JSON.parse(raw));

  return (
    <main>
      <h1>📮 お客さまの声</h1>
      <ul>
        {reviews.map((r) => (
          <li key={r.id}>
            {"⭐".repeat(r.stars)} {r.comment} — {r.author}
          </li>
        ))}
      </ul>
    </main>
  );
}
```

このコードの何が革命なのか、[React 第 14 章の劇評コーナー](../../05-react-fable-101/chapters/14_data_fetching.md)と
並べると一目瞭然です:

| | React(客席で fetch) | Next.js(厨房で調理) |
|---|---|---|
| データ取得 | useEffect + fetch + ignore 旗 | **ただの await** |
| 3 局面(loading/error)管理 | union 型 + switch で自前管理 | この章では不要(待つのはサーバー。第 9 章で洗練) |
| 競合状態(race) | 手動で防衛 | 発生しない(リクエストごとに 1 回実行) |
| DB・ファイル・秘密鍵へのアクセス | 不可能(ブラウザだから) | **直接可能** |
| このコードの JS は客に届く? | 届く(バンドルに入る) | **届かない**(結果の HTML だけ) |

useEffect・useState・ignore 旗・ローディング分岐——あの章の苦労の大半は
「**データがある場所(サーバー)と画面を作る場所(ブラウザ)が離れていた**」ことが
原因でした。画面を作る場所をデータの隣(厨房)に移せば、問題ごと消える。
これが Server Components の発想です。

> ⚙️ **厨房の真実 — 届いているのは「盛り付け済みの皿」**
>
> Server Component の実行結果としてブラウザに届くのは、HTML と、**RSC ペイロード** と
> 呼ばれる「レンダリング済みの設計図データ」です([React 第 1 章](../../05-react-fable-101/chapters/01_jsx.md)で
> 学んだ「JSX の正体 = 設計図オブジェクト」の直列化版だと思ってください)。
>
> つまり客席に届くのは **完成した料理(結果)** であって、**レシピ(コンポーネントの
> コード)ではありません**。`zod` も `node:fs` も `menuItems` の全データも、客の端末には
> 送られない——これは体感速度だけでなく、**バンドルサイズとセキュリティ**(秘密の
> API キーやビジネスロジックが客に渡らない)の話でもあります。
> 第 1 章の「調理器具一式を客に持たせる店(CSR)」との対比が、ここで完全に閉じます。

## 制約 — 厨房には客がいない

タダ飯はありません。サーバーには **ブラウザがなく、操作する客もいません**。
したがって Server Component では次が使えません:

- **`useState` / `useEffect` などの状態系・副作用系フック** — 状態とは
  [「再上演をまたぐ記憶」](../../05-react-fable-101/chapters/05_state.md)でしたが、サーバーでの
  レンダリングはリクエストごとに 1 回きり。覚える「次回の上演」がありません
- **`onClick` などのイベントハンドラ** — クリックする人が厨房にはいません
- **`window`、`localStorage` などのブラウザ API** — 存在しません

```tsx
// ❌ Server Component でこれを書くと、エラーで止まる
export default function Bad() {
  const [count, setCount] = useState(0);   // Error: useState は Client Component 専用
  return <button onClick={() => setCount(count + 1)}>{count}</button>;
}
```

「じゃあ拍手カウンタ(React 第 5 章)はどう作るんだ?」——その答え(`"use client"` で
**客席側のコンポーネントを宣言する**)が次章です。Next.js のコンポーネント設計とは、
突き詰めれば **「どの部品を厨房に置き、どの部品を客席に置くか」の采配** なのです。

> 📜 **歴史の背景 — RSC は React 史上最大の賭け**
>
> Server Components は 2020 年末に React チームが発表した構想で、Next.js 13.4(2023)の
> App Router が最初の本格実装です。「サーバーで React を動かす」こと自体は SSR として
> 昔からありましたが、従来の SSR は「**全コンポーネント** をサーバーで一度描き、
> **同じコード全部** をブラウザにも送ってもう一度実行する(ハイドレーション)」方式でした。
> 料理を届けたのに、レシピと食材一式も同梱していたわけです。
>
> RSC の新しさは「**サーバー専用の部品と客席用の部品を、1 つのコンポーネントツリーの
> 中で混在させる**」こと。届けるレシピは客席用の部品のぶんだけで済みます。
> ただしこの転換は「React = ブラウザの UI ライブラリ」という 10 年の常識を壊すもので、
> 学習コストの増大やエコシステムの分断([react の overview 6.1/6.5](../../05-react-fable-101/language-overview/README.md))を
> 招いた、現在進行形の大論争でもあります。この教材が React 教材(全部客席)と分けて
> Next.js を教えているのは、まさにこの境界を曖昧にしないためです。

## ⚔️ 完成コード: お客さまの声ページ

`data/reviews.json` を用意します:

```json
[
  { "id": 1, "author": "常連のサトウ", "stars": 5, "comment": "カルボナーラの追いチーズは事件。" },
  { "id": 2, "author": "はらぺこ大学生", "stars": 4, "comment": "オムライス大盛り無料が神。" },
  { "id": 3, "author": "辛口グルメ隊", "stars": 3, "comment": "カレーは旨いが中辛のみは残念。" }
]
```

上の `ReviewsPage` を `app/reviews/page.tsx` に置き、NavBar にリンクを足せば完成です。
「ページのソースを表示」で、レビューの文面が **HTML に焼き込まれている** こと、
ブラウザに `reviews.json` への fetch が **存在しない**(Network タブ)ことを確認してください。

## 📝 今日の仕込み(演習)

1. `console.log` をトップページに仕込み、「ターミナルに出る/ブラウザに出ない」を確認してください。次章で "use client" を学んだら同じ実験をもう一度やります(伏線)。
2. `ReviewsPage` で `reviews.json` のパスをわざと間違えて、どんなエラーがどこに表示されるか観察してください(開発モードのエラー画面もサーバーのエラーを映しています)。
3. `data/reviews.json` の 1 件の `stars` を `"5"`(文字列)にして、zod の門番が実行時に止めることを確認してください。[React 第 14 章の演習](../../05-react-fable-101/chapters/14_data_fetching.md)と同じ実験を、今日は厨房側でやっている——実行場所が変わっても門番の作法は同じです。
4. (考察)「メニュー台帳に原価(cost)フィールドを追加したいが、客には絶対見せたくない」。Server Component の性質を踏まえると、これは安全にできるでしょうか?「JSX に書かなければ届かない」のか「import しただけで届く」のか、⚙️ の内容から推論してください(答え合わせは次章の境界の話で)。

---

次章、客席に降ります。拍手カウンタ的な「触れる UI」を Next.js でどう作るか——
`"use client"` という境界線と、厨房と客席の間で **props に何を載せられるか** という
シリアライズの掟です。 → [第6章 客席との境界線](06_use_client.md)
