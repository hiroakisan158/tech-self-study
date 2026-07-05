# 第16章 テストと千秋楽 — 卒業制作

## 🎭 今日のお話

千秋楽の朝です。Reactive Theater のシステムはひととおり完成しました。最後の仕事は
**「壊れていないことを、いつでも数秒で確認できる」** 状態にすること——テストです。

[TS 第 16 章](../../04-typescript-fable-101/chapters/16_final.md)で「型が形を守り、テストが
振る舞いを守る」と学びました。React ではそこに独特の問いが加わります:
**UI のテストとは、何を確かめることなのか?**

## 道具立て — Vitest + Testing Library

```bash
npm install -D vitest jsdom @testing-library/react @testing-library/user-event @testing-library/jest-dom
```

| 道具 | 役割 |
|---|---|
| **Vitest** | テストランナー([TS 第 16 章](../../04-typescript-fable-101/chapters/16_final.md)と同じ) |
| **jsdom** | Node の中に作る「偽のブラウザ環境」(DOM の模型) |
| **Testing Library** | コンポーネントを描画し、**ユーザーの目線で** 探して操作する道具 |
| **user-event** | クリックやタイピングを本物らしく再現する道具 |

`vite.config.ts` に 3 行足せば準備完了です:

```ts
/// <reference types="vitest/config" />
import { defineConfig } from "vite";
import react from "@vitejs/plugin-react";

export default defineConfig({
  plugins: [react()],
  test: {
    environment: "jsdom",                 // ブラウザ模型の中でテストを走らせる
    setupFiles: "./src/setupTests.ts",    // 中身は import "@testing-library/jest-dom"; の 1 行
  },
});
```

## ユーザー目線の哲学 — 「実装」ではなく「体験」をテストする

Testing Library には設計思想が刻印されています:

> **「テストがソフトウェアの使われ方に似ているほど、テストは信頼に足る」**

つまり、state の中身や関数の呼び出し回数(**実装の詳細**)を覗くのではなく、
**画面に何が見え、操作したら何が起きるか**(ユーザーの体験)だけを確かめます:

```tsx
// src/ApplauseMeter.test.tsx
import { describe, it, expect } from "vitest";
import { render, screen } from "@testing-library/react";
import userEvent from "@testing-library/user-event";
import { ApplauseMeter } from "./ApplauseMeter";   // 第 5 章の拍手メーター

describe("拍手メーター", () => {
  it("初期状態では 0 回と表示される", () => {
    render(<ApplauseMeter showTitle="ハムレット" />);
    expect(screen.getByText(/拍手 0 回/)).toBeInTheDocument();
  });

  it("拍手ボタンを押すと回数が増える", async () => {
    const user = userEvent.setup();
    render(<ApplauseMeter showTitle="ハムレット" />);

    await user.click(screen.getByRole("button", { name: "👏 拍手" }));
    await user.click(screen.getByRole("button", { name: "👏 拍手" }));

    expect(screen.getByText(/拍手 2 回/)).toBeInTheDocument();
  });
});
```

- `render` が jsdom の中に部品を描画し、`screen` がその画面を表します
- **要素は「ユーザーに見える手がかり」で探します**: `getByRole("button", { name: ... })`
  (役割と表示名)、`getByText`(表示文字列)、`getByLabelText`(フォームのラベル)。
  `id` や CSS クラスで探さないのは、それがユーザーに見えないもの=実装の詳細だからです
- state が 2 になったかは **見ません**。「拍手 2 回」と **表示されたか** を見ます。
  こうしておけば、`useState` を `useReducer` に書き換えても(実装を変えても)
  テストは緑のまま——**リファクタリングを恐れなくてよいテスト** になります

💡 `getByRole` の「role」は、スクリーンリーダー等の支援技術がボタンや見出しを
識別するための仕組み(アクセシビリティ)と同じものです。**role で探せない UI は、
支援技術からも見えにくい UI** ——テストの書きやすさがアクセシビリティの検査を
兼ねる、良くできた仕掛けです。

## 何をどの層でテストするか — 三層の分業

16 章ぶんの部品には、それぞれ適したテストの層があります:

| 層 | 対象 | 道具 | 例 |
|---|---|---|---|
| 純粋ロジック | reducer、集計関数、バリデーション | Vitest のみ(描画不要・最速) | `theaterReducer` の遷移規則([第 13 章](13_reducer.md)) |
| コンポーネント | 表示と操作の対応 | + Testing Library | フォーム入力→ボタン活性化([第 6 章](06_forms.md)) |
| 非同期連携 | 通信の 3 局面 | + fetch のモック | 劇評の loading → success([第 14 章](14_data_fetching.md)) |

**下の層でテストできるものは下の層で。** [reducer を純粋関数として設計した](13_reducer.md)
配当がここで出ます——UI を一切描画せず、進行台本の全遷移を高速に検証できます:

```tsx
// src/theaterReducer.test.ts — 描画ゼロ、ミリ秒で全遷移を検証
import { describe, it, expect } from "vitest";
import { theaterReducer, type TheaterState } from "./theaterReducer";

const closed: TheaterState = { phase: "closed", audience: 0, applause: 0 };

describe("公演進行台本", () => {
  it("開場前に幕は上がらない", () => {
    expect(theaterReducer(closed, { type: "curtain_up" })).toEqual(closed);
  });

  it("開場 → 入場 → 開演の正規ルート", () => {
    let s = theaterReducer(closed, { type: "doors_opened" });
    s = theaterReducer(s, { type: "guest_entered", count: 3 });
    s = theaterReducer(s, { type: "curtain_up" });
    expect(s).toEqual({ phase: "performing", audience: 3, applause: 0 });
  });
});
```

非同期の層では、`fetch` を偽物に差し替えて 3 局面を再現します:

```tsx
// src/ReviewCorner.test.tsx(抜粋)
import { vi } from "vitest";

it("取得成功で劇評が表示される", async () => {
  vi.stubGlobal("fetch", vi.fn().mockResolvedValue({
    ok: true,
    json: async () => [{ id: 1, critic: "アオヤマ", stars: 5, comment: "圧巻" }],
  }));

  render(<ReviewCorner />);
  expect(screen.getByText(/取り寄せ中/)).toBeInTheDocument();          // loading 局面
  expect(await screen.findByText(/圧巻/)).toBeInTheDocument();         // success 局面
  //            ~~~~~~~~~ findBy* は「現れるまで待つ」非同期版の探し方
});
```

## 🏰 千秋楽 — Reactive Theater 全景

16 章で建てた劇場の見取り図です。各部品がどの章の成果か、思い出しながら眺めてください:

```
reactive-theater/
├── package.json / vite.config.ts        # 第16章: テスト設定込み
├── public/reviews.json                  # 第14章: 城壁の外のデータ
└── src/
    ├── main.tsx                         # エントリポイント(Vite が生成)
    ├── App.tsx                          # 座長(第8章: state の置き場所)
    ├── ThemeContext.tsx                 # 第12章: 劇場全体の照明
    ├── theaterReducer.ts                # 第13章: 進行台本(純粋関数)
    ├── theaterReducer.test.ts           # 第16章: 台本の全遷移テスト
    ├── hooks/
    │   ├── useCountdown.ts              # 第11章: 時を刻む巻物
    │   └── useLocalStorage.ts           # 第11章: 保存する巻物
    ├── components/
    │   ├── BoxOffice.tsx                # 第6章: 制御されたフォーム
    │   ├── ReservationBook.tsx          # 第7章: 不変更新の台帳
    │   ├── LobbyBoard.tsx               # 第8章: props で連動する掲示板
    │   ├── SeatMap.tsx                  # 第15章: memo された座席表
    │   └── ReviewCorner.tsx             # 第14章: 3局面 + zod の門番
    └── *.test.tsx                       # 第16章: ユーザー目線のテスト
```

そして卒業証書として、この教材の背骨を一枚に:

| React の原理 | 学んだ章 |
|---|---|
| UI = f(state) — 画面は状態の宣言的な関数 | 1 |
| データは下り(props)、出来事は上り(コールバック) | 2, 4, 8 |
| リストには本人確認(key)を | 3, 10 |
| state は「再上演を予約する記憶箱」。スナップショットで読む | 5 |
| 真実は 1 箇所に(制御されたフォーム、リフトアップ) | 6, 8 |
| 変更は複製で。React は参照しか見ない | 7, 15 |
| 外の世界との同期だけが effect。始めたら片付ける | 9, 14 |
| 稽古(レンダー)と本番(コミット)は別物 | 10 |
| ロジックは巻物(カスタムフック)に、変更は台本(reducer)に | 11, 13 |
| 放送(Context)は広く静かな情報だけ | 12 |
| 境界から来るデータは門番を通す | 14 |
| 最適化は測ってから。最善の最適化は構成 | 15 |
| テストは実装ではなく体験を確かめる | 16 |

## 🎓 次の冒険 — Next.js へ

この教材の React は、すべてブラウザの中で動いていました(クライアントサイド)。
実務ではさらに、次の問いが待っています——初回表示を速くするには?検索エンジンに
中身を読ませるには?ページ遷移とデータ取得を整理するには?

その答えをまとめて提供するのが **Next.js** です。あなたはもう入口に立っています:

- **Server Components**([第 10 章の📜](10_rendering.md)で予告)— 「稽古場をサーバーに置く」。
  データ取得([第 14 章](14_data_fetching.md)の苦労)の多くがサーバー側で消えます
- **ルーティング** — ファイルを置けばページになる(App Router)
- **Server Actions** — フォーム([第 6 章](06_forms.md))の送信先を関数として書く

公式チュートリアル [nextjs.org/learn](https://nextjs.org/learn) が最良の次の一歩です。
そこで見る「見慣れない魔法」の下では、必ずこの教材の 16 章が動いています。
**どこまでが React で、どこからが Next の魔法か** を見分けられること——それが
この劇場で身につけた、いちばんの財産です。

## 📝 最後の舞台稽古(卒業演習)

1. `BoxOffice`(第 6 章)のテストを書いてください: 「名前が空なら予約ボタンが押せない」「入力して押すと完了メッセージが出る」の 2 本。`getByLabelText` と `toBeDisabled()` が役立ちます。
2. `ReviewCorner` の **エラー局面** のテストを書いてください(`mockResolvedValue({ ok: false, status: 500, ... })`)。「もう一度取り寄せる」ボタンの表示確認まで。
3. わざと `theaterReducer` の遷移ガードを 1 つ消して、テストが赤くなることを確認してください([赤を見るのも卒業体験](../../04-typescript-fable-101/chapters/16_final.md)です)。
4. 卒業制作: 劇場に好きな新機能を 1 つ、**型 → 状態設計(どこに・useState か useReducer か)→ 実装 → テスト** の順で追加してください。設計から品質保証まで通せたら、あなたはもう立派な舞台監督です。

---

🎭 **Reactive Theater、千秋楽です。満場の拍手を——そして、次の舞台(Next.js)でお会いしましょう!** 👏👏👏
