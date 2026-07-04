# 第16章 卒業制作 — テストと完成

## 🍺 今日のお話

隣町への出荷を前に、ギルド長が最後の注文を出しました。「動くことを **証明** してくれ。
あなたが引退しても、次の受付係が安心して改修できるように」。

型検査は「ありえない書き方」を弾いてくれますが、「計算が正しいか」「状態遷移の順序が
守られるか」までは知りません。**型が守るのは形、テストが守るのは振る舞い** です。
最終日はテストを書き、16 章の旅の成果を一つのシステムに束ねます。

## Vitest — テストの道具

```bash
npm install -D vitest
```

`package.json` に追加:

```json
  "scripts": {
    "test": "vitest run",
    "test:watch": "vitest"
  }
```

Vitest は TypeScript をそのまま読めるため、追加設定なしで始められます
(前章の地図で言うと「テスト」役の現在の定番です。文法はほぼ Jest 互換なので、
古い記事の Jest の知識もだいたいそのまま通用します)。

## はじめてのテスト

テストファイルは対象の隣に `○○.test.ts` として置きます。

```typescript
// guild/src/treasury.test.ts
import { describe, it, expect, beforeEach } from "vitest";
import { deposit, withdraw, balance, reset } from "./treasury.js";

describe("金庫", () => {
  beforeEach(() => {
    reset(100);                       // 各テストの前に残高 100 に初期化
  });

  it("入金すると残高が増える", () => {
    deposit(50);
    expect(balance()).toBe(150);
  });

  it("残高の範囲内なら出金できる", () => {
    expect(withdraw(30)).toBe(true);
    expect(balance()).toBe(70);
  });

  it("残高を超える出金は拒否され、残高は変わらない", () => {
    expect(withdraw(999)).toBe(false);
    expect(balance()).toBe(100);      // 👈 失敗時に「何も起きていない」ことまで確認する
  });
});
```

- **describe** で対象ごとに束ね、**it** に「日本語で読める仕様」を書きます。
  テスト名は「〜すると〜になる」——**仕様書として読める**ことが良い名前の条件です
- **expect(実際).toBe(期待)** が検証の基本形。`.toEqual`(オブジェクトの中身比較)、
  `.toContain`、`.toThrow` などが頻出です
- **beforeEach** で毎回まっさらな状態から始めます。テスト同士が影響し合わないことは
  鉄則です(そのために `treasury.ts` に `reset` 関数を足しました——テストしやすさの
  ための改修は、たいてい設計自体も良くします)

```bash
npm run test
```

```
 ✓ guild/src/treasury.test.ts (3 tests) 
   ✓ 金庫 > 入金すると残高が増える
   ✓ 金庫 > 残高の範囲内なら出金できる
   ✓ 金庫 > 残高を超える出金は拒否され、残高は変わらない
```

## 何をテストするか — 型とテストの分業

16 章の学びがここで合流します。私たちのシステムには 3 種類の守りがあり、
それぞれ守備範囲が違います:

| 守り | 守備範囲 | 例 |
|---|---|---|
| **型**(第 1〜13 章) | ありえない形・操作を書けなくする | 「受注者のいない完了」は定義不能 |
| **門番**(第 14 章) | 城壁の外から来るデータの検問 | 不正な JSON の受け入れ拒否 |
| **テスト**(本章) | 城壁の中の **振る舞い** の正しさ | 昇格の閾値、税計算、遷移の順序 |

だからテストは「型が既に保証していること」には書きません。`takeQuest("文字列")` が
弾かれるかのテストは不要です(コンパイルが通らないのだから)。書くべきは
**型では表現しきれなかったルール** です:

```typescript
// guild/src/quests.test.ts
import { describe, it, expect } from "vitest";
import { postQuest, takeQuest, completeQuest } from "./quests.js";

describe("依頼の状態遷移", () => {
  it("受付中 → 受注 → 完了 の正規ルートが通る", () => {
    const q = postQuest("薬草採取", 30);
    expect(takeQuest(q, "カイ")).toBe(true);
    expect(completeQuest(q, "6月30日")).toBe(true);
    expect(q.status).toEqual({ state: "done", by: "カイ", completedOn: "6月30日" });
  });

  it("受注を飛ばした完了は拒否される", () => {
    const q = postQuest("看板修理", 15);
    expect(completeQuest(q, "6月30日")).toBe(false);
    expect(q.status.state).toBe("open");
  });

  it("二重受注は拒否される", () => {
    const q = postQuest("護衛任務", 90);
    takeQuest(q, "カイ");
    expect(takeQuest(q, "リタ")).toBe(false);
    expect(q.status).toEqual({ state: "taken", by: "カイ" });
  });
});
```

門番のテストには、非同期(第 12 章)がそのまま活きます:

```typescript
// guild/src/gatekeeper.test.ts
import { describe, it, expect } from "vitest";
import { writeFile, rm } from "node:fs/promises";
import { importQuests } from "./gatekeeper.js";

describe("門番", () => {
  it("正しい依頼書は受け入れる", async () => {
    await writeFile("test_ok.json", JSON.stringify([{ id: 1, title: "薬草採取", reward: 30 }]));
    const quests = await importQuests("test_ok.json");
    expect(quests).toHaveLength(1);
    await rm("test_ok.json");
  });

  it("reward が文字列の依頼書は拒否する", async () => {
    await writeFile("test_bad.json", JSON.stringify([{ id: 2, title: "退治", reward: "応相談" }]));
    await expect(importQuests("test_bad.json")).rejects.toThrow("様式が不正");
    await rm("test_bad.json");
  });
});
```

## 🏰 完成 — Typed Tavern ギルド管理システム

最終的なプロジェクトの全景です。各棟がどの章の成果か、確かめてください:

```
ts-guild/
├── package.json            # 第6・15章: 登記簿とスクリプト
├── package-lock.json
├── tsconfig.json            # 第6・15章: strict な検査設定
├── .gitignore               # node_modules/ と dist/
└── guild/src/
    ├── main.ts              # 受付窓口(エントリポイント)
    ├── quests.ts            # 第5・6・13章: 依頼台帳。判別可能 union の状態機械
    ├── quests.test.ts       # 第16章: 状態遷移のテスト
    ├── treasury.ts          # 第6章: 金庫。カプセル化
    ├── treasury.test.ts     # 第16章: 金庫のテスト
    ├── adventurers.ts       # 第7章: 冒険者クラス。# による本物の非公開
    ├── vault.ts             # 第8章: ジェネリック保管庫
    ├── report.ts            # 第9章: map/filter/reduce の月次報告
    ├── bell.ts              # 第10章: this を克服した閉店の鐘
    ├── owl.ts               # 第11・12章: async/await のフクロウ便
    ├── gatekeeper.ts        # 第14章: zod による実行時検証
    └── gatekeeper.test.ts   # 第16章: 門番のテスト
```

そして毎日の呪文は 4 つに集約されました:

```bash
npm run dev      # 開発: 保存のたびに自動再実行
npm run check    # 型検査: 形の正しさ
npm run test     # テスト: 振る舞いの正しさ
npm run build    # 出荷: 型を剥がして JavaScript へ
```

## 🎓 卒業 — 旅の振り返りと、次の冒険

お疲れさまでした。最後に、この教材の合言葉「**TS で書き、JS で理解する**」を
一枚の表にして卒業証書とします:

| JavaScript の性質(歴史) | TypeScript の手当て | 学んだ章 |
|---|---|---|
| 10 日間設計・動的型付け | 型注釈と推論、コンパイル時検査 | 1 |
| `==` の暗黙変換、number 一本 | `===` 徹底 + 型が異種比較を拒否 | 2 |
| 宣言なしで作れる野生のオブジェクト | interface と構造的型付け | 3 |
| 関数が第一級の値(Scheme の血) | 関数の型、文脈からの推論 | 4 |
| 「無」が 2 つある混沌 | union と narrowing による null 安全 | 5 |
| モジュールなしで生まれた言語 | ESM + tsconfig の整備 | 6 |
| class の下で動くプロトタイプ | 型付き class、`#` と `private` | 7 |
| 何でも入る入れ物 | ジェネリクスで型を保つ | 8 |
| 破壊/非破壊が混在する配列 API | イミュータブルな流儀 + 型の追跡 | 9 |
| 呼ばれ方で変わる this | アロー関数と this 型注釈 | 10 |
| シングルスレッド + イベントループ | (これは JS の設計そのものを理解する章) | 11 |
| コールバック地獄の歴史 | `Promise<T>` と async/await | 12 |
| 野生のコードの表現力 | 型から型を導く魔導書 | 13 |
| 型は実行時に消える | unknown・型ガード・zod の門番 | 14 |
| 公式ツール不在の生態系 | 役割で理解する道具の地図 | 15 |
| —— | 型が形を、テストが振る舞いを守る | 16 |

**次の冒険への道標:**

- **ブラウザと DOM** — 同じ言語がブラウザでは「画面を操作する」仕事をします。
  `document.querySelector` と `addEventListener` を触ると、第 4 章のコールバックと
  第 11 章のイベントループが「画面」という舞台で再演されます
- **React** — [react.dev](https://react.dev) の Learn から。コンポーネントは第 4 章の関数、
  props は引数、状態更新は第 9 章のイミュータブルな流儀、`useEffect` の背後には
  第 11 章のイベントループ——**この教材の知識がそのまま土台になります**
- **Next.js** — React の後に [nextjs.org/learn](https://nextjs.org/learn) へ。サーバーと
  ブラウザの境界を越えるとき、第 14 章の「城壁と門番」の感覚が効いてきます
- **姉妹教材** — [Python](../../python-fable-101/README.md) と [Go](../../go-fable-101/README.md) の
  fable-101、そして各言語の [language-overview](../../python-fable-101/language-overview/README.md) を
  読み比べると、「言語の設計はすべてトレードオフ」という視界が開けます

## 📝 最後の受付業務(卒業演習)

1. `adventurers.ts` の昇格ロジック(exp 100 で silver、300 で gold)のテストを自力で書いてください。「ちょうど 100」「一気に 450」の境界も忘れずに。
2. `report.ts` の `monthlyReport` をテストしやすいように「データを引数で受け取る純粋な関数」に改修し、テストを書いてください(グローバル状態に依存しない関数はテストが楽——という体験が目的です)。
3. わざと `completeQuest` のバグ(受注チェックの削除)を仕込んで `npm run test` が赤くなることを確認してください。**テストは「壊したら気づける」ためにある**——赤を見るのも大事な卒業体験です。
4. システムに好きな新機能を 1 つ、**型 → 実装 → テスト** の順で追加してください。あなたはもう、設計図から品質保証まで一人でこなせる受付係——いえ、ギルドマスターです。

---

⚔️ **Typed Tavern は本日も営業中。良い旅を、ギルドマスター!** 🍺
