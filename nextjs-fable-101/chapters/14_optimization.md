# 第14章 店の外観を磨く — 画像・フォント・SEO の最適化

## 🍽️ 今日のお話

料理写真をページに載せたところ、常連さんから「スマホだと表示が遅い」、
グルメブロガーからは「スクロール中に写真がガタッとずれる」と指摘されました。

味(機能)が良くても、外観と提供速度で客足は変わります。今日は Next.js が
**フレームワークの立場だからこそ自動化できる** 3 つの磨き込み——画像・フォント・
メタデータ——を学びます。新しい概念は少なめ、効果は大きめの章です。

## next/image — 写真の給仕を任せる

素の `<img>` で大きな写真を置くと、3 つの問題が起きます:
①原寸のまま配信されて重い(スマホに 4000px の写真を送る)、②画面外の写真まで
最初に全部読み込む、③読み込み完了時にレイアウトがガタッとずれる(あのブロガーの
指摘。CLS: Cumulative Layout Shift と呼ばれる、Google の順位指標にもなる問題です)。

`next/image` の `<Image>` に置き換えると、3 つとも自動で解決します:

```tsx
// app/menu/[id]/page.tsx に料理写真を追加
import Image from "next/image";

<Image
  src={`/photos/${item.id}.jpg`}     // public/photos/ に置いた写真
  alt={`${item.name} の写真`}
  width={800}
  height={600}
  priority={false}                    // 画面上部の重要画像だけ true にする
/>
```

> ⚙️ **厨房の真実 — Image が裏でやっていること**
>
> - **リサイズと形式変換**: リクエストした端末の画面幅に合わせ、サーバー側で縮小し、
>   対応ブラウザには WebP/AVIF(高圧縮形式)へ変換して配信します。スマホに 4000px の
>   原寸が送られることはなくなります
> - **遅延読み込み(lazy loading)**: 画面外の写真は、スクロールで近づくまで
>   取得しません
> - **場所の予約**: `width` / `height` の比率から表示領域を **先に確保** するので、
>   読み込み完了時のガタつき(CLS)が起きません
>
> どれも手作業でも可能な最適化です(そして手作業では誰もやり切れなかった最適化です)。
> `width`/`height` が必須なのは「ガタつき防止のために場所の予約が必要だから」——
> 制約には理由がある、という好例です。

## next/font — 看板文字のちらつきを消す

Web フォント(Google Fonts など)を普通に `<link>` で読み込むと、
**フォント到着前後で文字が描き直され、一瞬ちらつきます**(FOUT)。さらに
外部サーバーへの接続そのものが遅延源になります。`next/font` はフォントファイルを
**ビルド時に自分のプロジェクトへ取り込み**、自己ホストします:

```tsx
// app/layout.tsx
import { Noto_Sans_JP } from "next/font/google";

const notoSansJP = Noto_Sans_JP({
  subsets: ["latin"],
  weight: ["400", "700"],
});

export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html lang="ja" className={notoSansJP.className}>
      <body>...</body>
    </html>
  );
}
```

- 実行時に Google のサーバーへのアクセスは **発生しません**(ビルド時に取得済み)。
  速度・プライバシー・可用性の三得です
- サイズ調整用の CSS も自動生成され、ちらつきが抑えられます

💡 `Noto_Sans_JP(...)` を **モジュールのトップレベルで** 呼んでいることに注意
([コンポーネントの中ではない](../../react-fable-101/chapters/05_state.md))。これは
「ビルド時に 1 回だけ評価される場所」であり、フォント取り込みがビルド時の仕事で
あることと対応しています。

## メタデータの仕上げ — OGP という「店の顔写真」

[第 3 章で title と description](03_layouts.md) は設定しました。仕上げは **OGP
(Open Graph Protocol)**——SNS やチャットにリンクを貼ったときに出る、あの
プレビューカードです:

```tsx
// app/layout.tsx の metadata を拡張
export const metadata: Metadata = {
  title: { template: "%s | Bistro Next", default: "Bistro Next" },
  description: "その場で調理してお届けする、心づくしの食堂",
  openGraph: {
    title: "Bistro Next",
    description: "その場で調理してお届けする、心づくしの食堂",
    images: ["/photos/storefront.jpg"],       // シェア時に出る店の顔写真
    locale: "ja_JP",
    type: "website",
  },
};
```

さらに、検索エンジン向けの案内図 2 点もファイル規約で生成できます:

```ts
// app/sitemap.ts — 全ページの地図(検索エンジンの巡回用)
import type { MetadataRoute } from "next";
import { menuItems } from "../data/menu";

export default function sitemap(): MetadataRoute.Sitemap {
  const base = "https://bistro-next.example.com";
  return [
    { url: base },
    { url: `${base}/menu` },
    ...menuItems.map((item) => ({ url: `${base}/menu/${item.id}` })),
  ];
}
```

```ts
// app/robots.ts — 巡回ロボットへのお願い
import type { MetadataRoute } from "next";

export default function robots(): MetadataRoute.Robots {
  return {
    rules: { userAgent: "*", allow: "/", disallow: "/admin/" },   // 裏口は載せない
    sitemap: "https://bistro-next.example.com/sitemap.xml",
  };
}
```

台帳から sitemap を生成しているので、**料理を増やせば地図も自動で増えます**
([「データと画面の一致」](../../react-fable-101/chapters/03_lists.md)が SEO にまで貫通)。
`/sitemap.xml` と `/robots.txt` にアクセスして出力を確認してください。

> 📜 **歴史の背景 — 「速さ」が検索順位になった日**
>
> Google は 2021 年、**Core Web Vitals**(LCP: 主要コンテンツの表示速度、CLS: ガタつき、
> INP: 操作への反応)を検索順位の要素に組み込みました。「速い店が上に載る」時代の
> 到来です。この章の道具は全部この文脈にあります——Image は LCP と CLS、font は
> CLS、そもそもの [SSR/SSG(第 7 章)](07_rendering.md)は LCP のためでした。
>
> そしてこれが、[React 単体(CSR)ではなくフレームワークが推奨される](../../react-fable-101/language-overview/README.md)
> 実務的な理由の核心です。CSR の白い画面は、いまや UX の問題であると同時に
> **集客(SEO)の問題** なのです。第 1 章の「グルメサイトに載らない店」の伏線が、
> 数値指標の話としてここで回収されます。

## 計測 — 磨けたかどうかは目視ではなく数字で

磨き込みの効果は Lighthouse(Chrome 内蔵の採点ツール)で測ります:

```bash
npm run build && npm run start   # 計測は必ず本番モードで(dev は遅くて当然)
```

Chrome の開発者ツール → Lighthouse タブ → 「パフォーマンス」で分析。
Performance / Accessibility / Best Practices / SEO の 4 項目が採点され、
LCP や CLS の実測値と改善提案が出ます。
[React 第 15 章の「まず測る」](../../react-fable-101/chapters/15_performance.md)と同じ規律です
——あちらは「再レンダリングの無駄」を Profiler で、こちらは「配信と表示の無駄」を
Lighthouse で。**層が違えば測る道具も違う** ということも、3 階建ての理解の一部です。

## 📝 今日の仕込み(演習)

1. フリー素材の料理写真を 3 枚 `public/photos/` に置き、`<img>` 版と `<Image>` 版で Network タブを比較してください(転送サイズと、画面外画像の読み込みタイミング)。
2. 開発者ツールで回線を「Slow 4G」にし、`<img>` 版のガタつき(CLS)と `<Image>` 版の場所予約の違いを観察してください。
3. Lighthouse をトップページに対して実行し、スコアと指摘事項を記録してください。指摘のうち 1 つを直して再計測——「測る → 直す → 測る」を一周すること。
4. `/menu/[id]` の `generateMetadata`(第 4 章)に `openGraph.images` を追加し、料理ごとに違う顔写真でシェアされるようにしてください。

---

次章、いよいよ店の外へ——`next build` が吐き出すものの正体を確かめ、
世界に公開(デプロイ)します。「Vercel に置くと何が起き、自前サーバーだと何が
必要か」まで。 → [第15章 のれん分け](15_build_deploy.md)
