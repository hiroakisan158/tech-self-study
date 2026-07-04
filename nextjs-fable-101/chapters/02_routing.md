# 第2章 間取りが URL になる — ファイルベースルーティング

## 🍽️ 今日のお話

Bistro Next に部屋を増やします。お品書きページ(`/menu`)、店舗案内(`/about`)、
アクセス(`/about/access`)。

React 単体の世界なら、ここでルーティングライブラリを選定し(React Router?
TanStack Router?)、ルート表を書き、`<Routes>` を設定し……という工程でした。
Next.js の答えは拍子抜けするほど物理的です: **フォルダを掘ってください。それが URL です。**

## 規約 — フォルダ = URL、page.tsx = その部屋の中身

```
app/
├── page.tsx              → /            (トップ)
├── menu/
│   └── page.tsx          → /menu        (お品書き)
└── about/
    ├── page.tsx          → /about       (店舗案内)
    └── access/
        └── page.tsx      → /about/access(アクセス)
```

ルールは 2 つだけです:

1. **`app/` 以下のフォルダ構造が、そのまま URL のパス構造になる**
2. **その URL でページを表示するには、フォルダに `page.tsx` を置き、
   コンポーネントを `export default` する**

さっそくお品書きページを作ります:

```tsx
// app/menu/page.tsx
const menuItems = [
  { id: "carbonara", name: "追いチーズカルボナーラ", price: 1200 },
  { id: "omurice", name: "とろとろオムライス", price: 980 },
  { id: "curry", name: "牛すじ欧風カレー", price: 1100 },
];

export default function MenuPage() {
  return (
    <main>
      <h1>📖 お品書き</h1>
      <ul>
        {menuItems.map((item) => (
          <li key={item.id}>
            {item.name} — {item.price.toLocaleString()} 円
          </li>
        ))}
      </ul>
    </main>
  );
}
```

保存して http://localhost:3000/menu を開けば、もう表示されます。ルート表への登録は
不要です。**map と key は [React 第 3 章](../../react-fable-101/chapters/03_lists.md)の
知識そのまま**、Next が足したのは置き場所の規約だけ——という階層の切り分けを、
今日も確認してください。

> 📜 **歴史の背景 — 「設定より規約」**
>
> ファイルベースルーティングは PHP の時代(ファイルを置けば URL になる)への
> 先祖返りでもあり、Ruby on Rails が広めた **「設定より規約(Convention over
> Configuration)」** の思想でもあります。ルート表という「設定」は自由ですが、
> チームの数だけ流儀が生まれ、URL とコードの対応を追うのに脳内変換が要ります。
> 「フォルダ = URL」という規約は自由を削る代わりに、**誰のプロジェクトでも
> `app/` を見れば全ページが一覧できる** という読みやすさを買っています。
> [gofmt が書式の自由を削って読みやすさを買った](../../go-fable-101/language-overview/README.md)のと
> 同じ取引です。
>
> なお `page.tsx` 以外にも、`layout.tsx`(第 3 章)、`loading.tsx` / `error.tsx`(第 9 章)、
> `route.ts`(第 11 章)など、**ファイル名そのものに役割を持たせる** のが App Router の
> 一貫した流儀です。

## Link — 部屋から部屋への廊下

ページ間の移動には `<a>` ではなく Next.js の **`<Link>`** を使います:

```tsx
// app/page.tsx に追記
import Link from "next/link";

export default function HomePage() {
  return (
    <main>
      <h1>🍽️ Bistro Next</h1>
      <p>心を込めて、その場で調理してお届けします。</p>
      <nav>
        <Link href="/menu">お品書き</Link> / <Link href="/about">店舗案内</Link>
      </nav>
    </main>
  );
}
```

> ⚙️ **厨房の真実 — `<a>` と `<Link>` の違い**
>
> 素の `<a href>` は **ページ全体の再読み込み** を起こします。せっかく届けた JS も
> React の state もすべて破棄され、客は毎回入店し直すことになります。
>
> `<Link>` は初回こそ普通の `<a>` として HTML に出力されますが(だから検索エンジンにも
> 読めます)、JS が有効になると挙動を乗っ取り、**必要な部分だけを取得して画面を
> 差し替えます**(クライアントサイドナビゲーション)。さらに、リンクが画面内に
> 見えた時点で遷移先を **先読み(prefetch)** しておくので、クリック時にはほぼ
> 待ちません。「初回はサーバー配膳の速さ、以降は SPA の滑らかさ」——
> 両取りこそが Next.js の基本戦略です。

## 現在地を知る — usePathname

ナビゲーションで「いま居るページ」を強調したいときは `usePathname` を使います:

```tsx
"use client";                              // ← この 1 行は第 6 章の主役。今日は写経で OK
import Link from "next/link";
import { usePathname } from "next/navigation";

const rooms = [
  { href: "/", label: "のれん" },
  { href: "/menu", label: "お品書き" },
  { href: "/about", label: "店舗案内" },
];

export function NavBar() {
  const pathname = usePathname();          // いまの URL パス(例: "/menu")

  return (
    <nav>
      {rooms.map((room) => (
        <Link
          key={room.href}
          href={room.href}
          style={{ fontWeight: pathname === room.href ? "bold" : "normal", marginRight: 12 }}
        >
          {room.label}
        </Link>
      ))}
    </nav>
  );
}
```

💡 このファイルは `app/` の外でも構いません(例: `components/NavBar.tsx`)。
URL になるのは **`page.tsx` という名前のファイルだけ** なので、部品は好きな場所に
置けます(`app/` 内に置いてもフォルダに page.tsx がなければ URL は生えません)。

## ⚔️ 完成コード: 今日の間取り

```
bistro-next/
├── app/
│   ├── page.tsx            # のれん(NavBar 付き)
│   ├── menu/page.tsx       # お品書き
│   └── about/
│       ├── page.tsx        # 店舗案内
│       └── access/page.tsx # アクセス
└── components/
    └── NavBar.tsx          # 現在地強調付きナビ
```

```tsx
// app/about/page.tsx
import Link from "next/link";

export default function AboutPage() {
  return (
    <main>
      <h1>🏠 店舗案内</h1>
      <p>Bistro Next は「その場で調理」と「作り置きの知恵」を両立する食堂です。</p>
      <Link href="/about/access">アクセスはこちら</Link>
    </main>
  );
}
```

```tsx
// app/about/access/page.tsx
export default function AccessPage() {
  return (
    <main>
      <h1>🚃 アクセス</h1>
      <p>App Router 線「ルート駅」徒歩 0 分(app フォルダの中にあります)</p>
    </main>
  );
}
```

## 📝 今日の仕込み(演習)

1. `/news`(お知らせ)ページを自分で作ってください。フォルダ作成 → page.tsx → export default、の 3 手です。
2. `page.tsx` を `Page.tsx` や `index.tsx` に改名すると何が起きるか試してください(規約の厳密さの確認)。
3. `<Link href="/menu">` を `<a href="/menu">` に変えて、遷移時の違いを観察してください(開発者ツールの Network タブで「ページ全体の再読み込み」が起きるかを見る)。
4. ブラウザで http://localhost:3000/kitchen(存在しない URL)を開いてください。Next.js 標準の 404 ページが出ます。`app/not-found.tsx` を作って、自分の店らしい 404(「その部屋はございません 🍜」)に差し替えてみましょう。

---

次章、どのページにも同じナビと店構えを出したくなりました。全ページに NavBar を
コピペ……はしません。**一度だけ描いて全部屋に効かせる** `layout.tsx` の出番です。
→ [第3章 店構えは一度だけ描く](03_layouts.md)
