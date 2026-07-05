# 第16章 テストと開店披露 — 卒業制作

## 🍽️ 今日のお話

開店披露の日です。最後の仕事は、[TS 教材](../../04-typescript-fable-101/chapters/16_final.md)・
[React 教材](../../05-react-fable-101/chapters/16_final.md)と同じく **「壊れていないことを
数秒で確認できる」** 状態を作ること。

ただし Next.js アプリには新しい事情があります。厨房(サーバー)と客席(ブラウザ)に
またがる機能——予約フォームから台帳更新、ページ作り直しまで——は、
**どちらか片方だけのテストでは検証できません**。今日は店の外から客として来店する
ロボット、**E2E(End-to-End)テスト** を主役に据えます。

## テストの三層 — 3 部作の総集編として

| 層 | 何を検証 | 道具 | 学んだ場所 |
|---|---|---|---|
| 純粋ロジック | 関数・スキーマ・reducer | Vitest | [TS 第 16 章](../../04-typescript-fable-101/chapters/16_final.md) |
| コンポーネント | 表示と操作の対応 | Vitest + Testing Library | [React 第 16 章](../../05-react-fable-101/chapters/16_final.md) |
| **E2E(本章)** | **厨房から客席までの全動線** | **Playwright** | 本章 |

下 2 層の書き方は、すでにあなたの手の中にあります(門番スキーマの単体テスト、
`OrderCounter` の操作テスト——書き方は前 2 教材とまったく同じです)。
Next.js 固有の課題は「ルーティング・Server Components・Actions・キャッシュが
絡んだ **統合的な振る舞い**」で、これは実際にアプリを起動してブラウザで
確かめるのが最も確実です。

💡 なお、async な Server Component の単体テストは、現時点ではツールの対応が
発展途上です(React の公式テスト道具がブラウザ前提で発達してきた歴史のため)。
**「Server Component の検証は E2E に寄せる」** のが現在の実務的な定石です——
道具の勢力図は変わるので、「層で考える」姿勢だけ持ち帰ってください。

## Playwright — ロボット客を雇う

```bash
npm init playwright@latest
# → tests/ フォルダ、playwright.config.ts、ブラウザ本体がセットアップされる
```

`playwright.config.ts` に「テスト前に店を開けておく」設定を入れます:

```ts
// playwright.config.ts(要点のみ)
export default defineConfig({
  testDir: "./tests",
  use: { baseURL: "http://localhost:3000" },
  webServer: {
    command: "npm run build && npm run start",   // 本番モードで検品(キャッシュ込みの姿)
    url: "http://localhost:3000",
    reuseExistingServer: !process.env.CI,
  },
});
```

最初のロボット客に、店内を一周してもらいます:

```ts
// tests/tour.spec.ts — 店内一周ツアー
import { test, expect } from "@playwright/test";

test("のれんからお品書き、個室まで辿り着ける", async ({ page }) => {
  await page.goto("/");
  await expect(page.getByRole("heading", { name: /Bistro Next/ })).toBeVisible();

  // ナビからお品書きへ(客がするのと同じ操作)
  await page.getByRole("link", { name: "お品書き" }).click();
  await expect(page).toHaveURL("/menu");

  // 料理の個室へ
  await page.getByRole("link", { name: /カルボナーラ/ }).click();
  await expect(page.getByRole("heading", { name: /カルボナーラ/ })).toBeVisible();
  await expect(page.getByText(/1,200 円/)).toBeVisible();
});

test("存在しない料理は 404 になる", async ({ page }) => {
  const response = await page.goto("/menu/sushi");
  expect(response?.status()).toBe(404);
  await expect(page.getByText(/その部屋はございません/)).toBeVisible();
});
```

```bash
npx playwright test           # ヘッドレス(画面なし)で全テスト
npx playwright test --ui      # 操作の様子を目で見ながらデバッグ
```

`getByRole` で探すのは [React 第 16 章の Testing Library と同じ哲学](../../05-react-fable-101/chapters/16_final.md)
——**実装ではなくユーザーの体験を確かめる**。Playwright は同じ問い合わせ言語を
本物のブラウザで実行している、と捉えれば、学ぶことはほとんど増えていません。

## 本丸 — 予約の全動線をテストする

この 3 部作の集大成にふさわしいテストは、第 8 章の予約機能です。
このテストが通るとき、何が検証されているかを数えてみてください:

```ts
// tests/reservation.spec.ts — 予約の全動線
import { test, expect } from "@playwright/test";

test("予約すると、完了メッセージと一覧の両方に反映される", async ({ page }) => {
  await page.goto("/reserve");

  const uniqueName = `テスト太郎_${Date.now()}`;      // 実行ごとに一意の名前
  await page.getByPlaceholder("お名前").fill(uniqueName);
  await page.getByRole("spinbutton").fill("3");
  await page.getByRole("button", { name: "予約する" }).click();

  // ① Server Action の戻り値が客席に表示される(useActionState)
  await expect(page.getByText(new RegExp(`${uniqueName} さま、3 名で承りました`))).toBeVisible();

  // ② revalidatePath による一覧の作り直しまで反映される
  await expect(page.getByRole("listitem").filter({ hasText: uniqueName })).toBeVisible();
});

test("名前が空だと門番に止められる", async ({ page }) => {
  await page.goto("/reserve");
  await page.getByRole("button", { name: "予約する" }).click();
  await expect(page.getByText(/お名前をご記入ください/)).toBeVisible();
});
```

1 本目のテストが緑になるとき、**フォーム描画(RSC)→ 送信(POST)→ zod の門番 →
台帳書き込み(Node.js)→ revalidatePath(キャッシュ)→ 再レンダリング → 客席での
表示** という、1F から 3F までの全階が一気通貫で検証されています。
E2E テストが「本数は少なくても価値が高い」と言われる理由です
(遅くて壊れやすくもあるので、**細かい分岐は下の層、幹だけ E2E** が配分の定石です)。

## 🏰 開店披露 — Bistro Next 全景

```
bistro-next/
├── app/
│   ├── layout.tsx / page.tsx        # 第1-3章: 店構えとのれん
│   ├── menu/ + [id]/                # 第2,4章: お品書きと個室(●SSG)
│   ├── reviews/                     # 第5章: 厨房でファイルから調理
│   ├── today/                       # 第7章: ISR の日替わり定食
│   ├── reserve/ + actions.ts        # 第8章: Server Actions の予約
│   ├── weekly/ + loading/error      # 第9章: ストリーミングとお詫び
│   ├── api/menu/ + api/orders/      # 第11章: 出前窓口
│   ├── staff-entrance/ + admin/     # 第12章: 関所の向こうの管理室
│   └── sitemap.ts / robots.ts       # 第14章: 検索エンジン向け案内図
├── components/                      # OrderCounter, Accordion, WeeklyBuzz...
├── data/                            # 台帳(menu.ts, reviews.json, ...)
├── lib/env.ts                       # 第13章: 起動時の検問所
├── middleware.ts                    # 第12章: 入口の案内係
└── tests/                           # 第16章: ロボット客
```

卒業証書として、3 部作の対応表を一枚に:

| 概念 | 1F: TypeScript | 2F: React | 3F: Next.js |
|---|---|---|---|
| 中心思想 | 型で「ありえない」を消す | UI = f(state) | できる限り作り置き |
| 実行場所 | Node.js / ブラウザ | ブラウザ | **厨房と客席の采配** |
| 状態 | 変数と型 | useState / reducer | サーバーの台帳 + revalidate |
| 非同期 | async/await・イベントループ | effect と 3 局面 | async コンポーネント・Suspense |
| 境界の守り | zod の門番 | props の型 | **全城門に門番**(params/form/API/env) |
| 合成 | 関数・ジェネリクス | children・カスタムフック | 客席の枠 × 厨房の中身 |
| テスト | Vitest(純粋関数) | Testing Library(体験) | Playwright(全動線) |

**縦に読めば各階の道具、横に読めば同じ思想の三変化** です。新しいフレームワークや
流行が来ても、この表の「行」は簡単には変わりません——道具の名前ではなく行の概念で
学んできたことが、あなたの資産です。

## 🎓 卒業後の地図 — ここから先の街

Bistro Next には、実務との差分がまだいくつかあります。次に学ぶ価値が高い順に:

1. **データベース** — `data/*.json` を本物の DB へ(第 15 章の演習 3 で痛感したはず)。
   TS 界の定番は **Prisma** / **Drizzle**(スキーマから型を導出——[zod と同じ
   「実行時の定義から型を導く」思想](../../04-typescript-fable-101/chapters/14_runtime_validation.md)です)
2. **認証** — 合言葉 Cookie を **Auth.js(NextAuth)** や **Clerk** へ。
   第 12 章の「通り道の地図」があれば、ドキュメントが素直に読めます
3. **UI の本格化** — CSS 設計(Tailwind CSS)、UI 部品集(shadcn/ui、Radix)
4. **クライアント側のサーバーデータ管理** — 高頻度に更新される画面では
   **TanStack Query** が定番([React 第 14 章の⚙️](../../05-react-fable-101/chapters/14_data_fetching.md)の続き)
5. **視野を広げる** — 同じ問題への別解(Remix / SvelteKit / Astro)を一つ触ると、
   Next.js の設計判断が相対化されて理解が深まります

## 📝 最後の仕込み(卒業演習)

1. 予約 E2E テストの「満席のとき予約できない」版を追加してください(台帳を満席近くまで埋める準備処理も含めて)。
2. 第 11 章の `POST /api/orders` を Playwright の `request` フィクスチャ(ブラウザなしで HTTP を叩く機能)でテストしてください: 正常 201 / 不正 400 の 2 本。
3. わざと `revalidatePath` を消して、E2E テストの ② が赤くなることを確認してください。**「キャッシュの戻し忘れをテストが捕まえる」**——第 10 章の苦労がテストで資産化される瞬間です。
4. 卒業制作: Bistro Next に好きな機能を 1 つ、**要件 → どの階の仕事か分解(1F/2F/3F)→ 型と門番 → 実装 → E2E** の順で追加してください。この分解が自力でできたら、もう「フルスタック開発者の見習い」ではありません。

---

🍽️ **Bistro Next、本日開店です。3 部作の長旅、お疲れさまでした——良い開発を、シェフ!** 👨‍🍳✨
