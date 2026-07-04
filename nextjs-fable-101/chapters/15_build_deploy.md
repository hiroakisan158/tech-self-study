# 第15章 のれん分け — ビルドの解剖とデプロイ

## 🍽️ 今日のお話

Bistro Next を世界に公開するときが来ました。ローカルの `npm run dev` は
あなたの PC の中だけの営業です。**デプロイ** とは、インターネット上のサーバーで
この店を 24 時間営業させること。

その前に、[TS 第 15 章で `tsc` の出力(dist/)を解剖した](../../typescript-fable-101/chapters/15_build_ecosystem.md)ように、
`next build` が何を作っているのかを見ておきます。**中身を知らないものは運用できません。**

## next build の解剖 — .next/ の中に店が丸ごと入っている

```bash
npm run build
```

ビルドは大きく 3 つの仕事をしています:

1. **翻訳と束ね** — TS → JS の翻訳([1F の知識](../../typescript-fable-101/chapters/15_build_ecosystem.md))、
   コンポーネントのバンドル、ページ単位のコード分割。このとき
   [厨房/客席の境界(第 6 章)](06_use_client.md)に沿って、**客席に送る JS と
   厨房専用コードを別々の袋に分けます**
2. **型検査と Lint** — エラーがあればビルドは止まります(不良品は出荷しない)
3. **作り置き** — [静的と判定された全ページ(第 7 章)](07_rendering.md)をこの場で
   レンダリングし、HTML として保存。あのビルドログの ○/●/ƒ 表は、この工程の報告書でした

結果はすべて `.next/` フォルダに入ります:

```
.next/
├── static/            # 客席へ配る袋(JS チャンク、CSS、フォント)— CDN 向き
├── server/            # 厨房のコード(Server Components、Actions、Route Handlers)
│   └── app/           #   ページごとの作り置き HTML と RSC ペイロードもここに
├── cache/             #   第 10 章で解剖した Data Cache などの棚
└── BUILD_ID           # このビルドの通し番号
```

つまり本番サーバーで動かすものの正体は:

```bash
npm run start   # 実体は next start — .next/ を読んで営業を始める Node.js サーバー
```

**Node.js プロセスが 1 つ立ち、静的ファイルを配り、動的ページを調理し、Action と
API を受け付ける**——それが Next.js アプリの本番の姿です。dev サーバーとの違いは
「事前に全部仕込み済みで、[棚(キャッシュ)が本気で働く](10_caching.md)」ことです。

## デプロイの選択肢 — どこで店を開くか

### 選択肢 1: Vercel(開発元のホスティング)

```bash
# GitHub にリポジトリを push して、vercel.com で Import するだけ
```

Next.js の開発元 Vercel のサービスに置くのが最短ルートです。`git push` するたびに
自動でビルド・公開され、静的ファイルは世界中の CDN に、動的処理はサーバーレス関数に
**自動で振り分けられます**。プレビュー環境(PR ごとの試食会場)も無料枠からあります。

> 📜 **歴史の背景 — 「フレームワークを配る会社」というビジネスモデル**
>
> Vercel は「Next.js を最も快適に動かせる場所」を売る会社です。オープンソースの
> フレームワークを無料で配り、その最適なホスティングで収益化する——この構図は
> 開発体験の投資を加速させた一方、[「Next.js の機能が Vercel 前提に寄っていく」
> という不信(react overview 6.5)](../../react-fable-101/language-overview/README.md)も生みました。
> 技術選定では「フレームワークの都合」と「その開発元の商売の都合」を分けて眺める
> 目も必要です。だからこそ、次の「自前で動かす」経験に価値があります。

### 選択肢 2: 自前のサーバー / コンテナ(セルフホスト)

`next start` が動く環境——つまり **Node.js が動く場所ならどこでも** 営業できます。
実務では Docker コンテナにするのが定番です:

```ts
// next.config.ts — コンテナ向けの出力設定
const nextConfig = {
  output: "standalone",   // 依存を同梱した「持ち運べる厨房」を .next/standalone に作る
};
export default nextConfig;
```

```dockerfile
# Dockerfile(要点のみ)
FROM node:22-alpine AS builder
WORKDIR /app
COPY . .
RUN npm ci && npm run build

FROM node:22-alpine
WORKDIR /app
COPY --from=builder /app/.next/standalone ./
COPY --from=builder /app/.next/static ./.next/static
COPY --from=builder /app/public ./public
ENV PORT=3000
CMD ["node", "server.js"]     # standalone が生成した最小サーバー
```

`output: "standalone"` は、`node_modules` の中から **実行に必要な分だけ** を抽出して
自己完結フォルダを作る設定です。[Go の「単一バイナリ」](../../go-fable-101/language-overview/README.md)ほどの
潔さはありませんが、思想は同じ「持っていけば動く」です。AWS・GCP・自宅サーバー、
どこでも同じコンテナが動きます。

### 選択肢 3: 完全な作り置き店(静的エクスポート)

全ページが静的にできるサイト(ブログ、会社案内)なら、`output: "export"` で
**ただの HTML/CSS/JS のフォルダ** を出力し、任意の静的ホスティング(GitHub Pages 等)に
置けます。ただし厨房を持たない屋台になるので、**Server Actions・動的レンダリング・
ISR・Image の自動変換は使えません**。Bistro Next は予約システム(第 8 章)があるので
この道は選べません——「どの機能がサーバー(厨房)を必要とするか」を言える、
というのがこの教材で養った判断力です。

| 選択肢 | 手間 | 自由度 | 向いている店 |
|---|---|---|---|
| Vercel 等のマネージド | 最小 | 低〜中 | 個人開発、スタートアップ、まず公開したい |
| セルフホスト(コンテナ) | 中 | 高 | 社内基盤がある、コストと主権を握りたい |
| 静的エクスポート | 小 | 最低 | 完全に静的なサイトのみ |

## 公開前チェックリスト — 3 部作の総点検

```bash
npm run check     # 型検査(1F: tsc --noEmit)
npm run lint      # Lint
npm run build     # ビルド(○/●/ƒ の再確認 — 意図しない ƒ がないか!)
npm run start     # 本番モードで一周(キャッシュ挙動・Lighthouse は必ずここで)
```

- **環境変数**([第 13 章](13_type_safety.md)): 本番の値をホスティング側に設定したか。
  `.env.local` は Git に入っていないか。`NEXT_PUBLIC_` に秘密が混ざっていないか
- **城門の門番**(第 4・8・11 章): params・フォーム・API body の zod は全箇所にあるか
- **多層防御**([第 12 章](12_middleware.md)): /admin は Middleware **と** ページ本体の
  両方で守られているか
- **キャッシュ**([第 10 章](10_caching.md)): 更新が反映されるべきページに revalidate の
  経路(時間 or イベント)はあるか

> ⚙️ **厨房の真実 — デプロイとは「棚の総入れ替え」でもある**
>
> 新しいビルドを公開すると、[Full Route Cache と Data Cache(第 10 章)](10_caching.md)は
> 新しい BUILD_ID の棚に切り替わります(= デプロイはキャッシュの全リセット)。
> 「反映されない!」と騒ぐ前にデプロイし直すと直る——のは、これが理由です。
> 逆に、頻繁にデプロイできない環境ほど revalidate 設計が効いてきます。

## 📝 今日の仕込み(演習)

1. `.next/server/app/` の中を覗いて、静的ページ(about など)の `.html` が **ビルド時点で既に存在する** ことを確認してください(第 7 章の「作り置き」の物証です)。
2. `output: "standalone"` を設定してビルドし、`.next/standalone/` フォルダだけを別の場所にコピーして `node server.js` で起動できることを確認してください(「持ち運べる厨房」の体感)。
3. 無料枠のあるホスティング(Vercel など)に実際にデプロイし、スマホからアクセスしてみてください。`data/*.json` への書き込み(第 8 章の予約)が本番でどうなるかも観察を——サーバーレス環境ではファイル書き込みが永続しないことに気づいたら、それが「実務では DB を使う」理由の体感です(卒業後の宿題として SQLite/Postgres への置き換えがあります)。
4. (考察)Bistro Next の全ページについて「Vercel / セルフホスト / 静的エクスポート」のどこまで対応できるかを表にしてください。静的エクスポートで諦める機能を列挙できれば、この章は修了です。

---

最終章です。開店披露の前に、店全体を **自動テスト** で検品します。ブラウザを
ロボットに操作させる E2E テストで「客の体験」ごと検証し、3 部作を締めくくります。
→ [第16章 テストと開店披露](16_final.md)
