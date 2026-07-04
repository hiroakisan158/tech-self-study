# 第13章 型の魔導書 — 高度な型操作

## 🍺 今日のお話

ギルドの書庫の最奥に、一冊の魔導書が眠っています。開くと「**型から型を生み出す術**」が
記されていました。「Quest の全項目を任意入力にした型」「Quest の項目名だけの型」——
既にある型を素材に、新しい型を **計算で** 導き出せるというのです。

なぜそんな術が要るのでしょう? 例えば「依頼の一部だけを修正する関数」を作るとき、
`Quest` とほぼ同じで全部が省略可能な型を **手書きで複製** したら、`Quest` に項目を
足すたびに複製も直す羽目になります(そして必ず直し忘れます)。
**型を派生させれば、元が変われば派生も自動で変わる。** これが今日の魔法の実用価値です。

## keyof と typeof — 素材の採取

```typescript
interface Quest {
  id: number;
  title: string;
  reward: number;
  done: boolean;
}

type QuestKey = keyof Quest;          // "id" | "title" | "reward" | "done"
//   ↑ プロパティ名たちが、文字列リテラル union として手に入る!

type RewardType = Quest["reward"];    // number(プロパティの型を取り出す)
```

`keyof` の実用例——「オブジェクトの指定プロパティを安全に読む」関数:

```typescript
function pluck<T, K extends keyof T>(items: T[], key: K): T[K][] {
  return items.map((item) => item[key]);
}

pluck(quests, "title");    // string[] — 戻り値の型まで正確!
pluck(quests, "reward");   // number[]
pluck(quests, "titel");    // ❌ コンパイルエラー: typo は QuestKey にない
```

もう一つの採取道具が `typeof`(型の文脈版)です。**値から型を吸い上げます**:

```typescript
const defaultQuest = { id: 0, title: "無題", reward: 0, done: false };
type QuestLike = typeof defaultQuest;   // 値の形から型を生成
```

💡 第 2 章で使った実行時の `typeof x === "number"` とは **別物** です(同じ字面で、
型の世界と値の世界に別々の住人がいます)。型の世界の記述はすべて実行時に消える——
第 1 章の原則をここでも思い出してください。

## Utility Types — 公式魔導書の頻出呪文

TypeScript には「型から型を作る」定番の術が最初から収録されています。

| 呪文 | 効果 | ギルドでの用途 |
|---|---|---|
| `Partial<T>` | 全プロパティを省略可能に | 一部だけの修正依頼 |
| `Pick<T, K>` | 指定プロパティだけ抜き出す | 掲示板の要約表示 |
| `Omit<T, K>` | 指定プロパティを除いた残り | 「id は自動採番なので受け取らない」 |
| `Readonly<T>` | 全プロパティを readonly に | 確定した台帳の凍結 |
| `Record<K, V>` | K の各値をキーに持つ辞書型 | ランクごとの昇格条件表(第 7 章で既出!) |
| `ReturnType<F>` | 関数の戻り値の型 | 他人の関数の結果を受ける変数 |

実戦での使いどころが最も多いのは `Partial` と `Omit` です:

```typescript
// 「id 以外を受け取って新規登録」— 第 4 章の postQuest が型で洗練される
function createQuest(input: Omit<Quest, "id">): Quest {
  return { id: nextId++, ...input };
}

// 「変えたい項目だけ渡して修正」— パッチ更新の定番形
function updateQuest(q: Quest, patch: Partial<Quest>): Quest {
  return { ...q, ...patch };    // スプレッドの後勝ちルールで patch が上書き
}

updateQuest(quest, { reward: 100 });          // ✅ 一部だけでいい
updateQuest(quest, { rewrd: 100 });           // ❌ typo は Partial でも通さない
```

`Quest` に項目を足しても、`createQuest` と `updateQuest` の型は **自動で追従** します。
手書き複製の未来のバグが、定義した瞬間に消えました。

## 種明かし — Mapped Types

`Partial` は組み込みの黒魔術ではありません。**自分で書けます**:

```typescript
type MyPartial<T> = {
  [K in keyof T]?: T[K];
  // 「T の各キー K について、同じ型 T[K] のプロパティを ? 付きで並べ直す」
};
```

`[K in ...]` は **型の世界の for ループ**(mapped type)です。`keyof` で鍵束を採取し、
1 本ずつ回して新しい型を組み立てる——魔導書の呪文の正体は、この小さな仕組みの応用です。

> 📜 **歴史の背景 — 型システムはなぜここまで強力になったのか**
>
> TypeScript の使命は「**既存の JavaScript の野生の書き方に、あとから型を付けきる**」こと
> でした(第 3 章)。JS の実世界のコードは「設定オブジェクトのキーで挙動が変わる」
> 「引数の型によって戻り値が変わる」など、静的型の常識を超えた書き方に満ちています。
> それを表現するために型の道具は増え続け、ついに **TypeScript の型システムはチューリング
> 完全**(理論上あらゆる計算が可能)になりました。型だけでチェスや SQL パーサを実装した
> 猛者もいます(実用性はゼロですが、表現力の証明として)。
>
> 実務での心得: 魔導書は **読める** ことが大事で、**唱えすぎない** ことも大事です。
> 凝りすぎた型はコンパイルを遅くし、同僚を置き去りにします。日常の 95% は
> union、interface、ジェネリクス、そして上の表の Utility Types で足ります。

## unknown と any — 「不明」の正しい扱い方

型の魔導書の最終ページには、対になる 2 つの型が記されています。

```typescript
let a: any = loadSomething();
a.definitely.not.exist();        // ✅ 通ってしまう(そして実行時に 💥)
const n: number = a;             // ✅ これも通る — any は検査の全面停止

let u: unknown = loadSomething();
u.anything();                    // ❌ エラー: 正体不明のものには触れない
if (typeof u === "string") {
  console.log(u.toUpperCase());  // ✅ narrowing で正体を確かめれば使える
}
```

- **`any`** = 「型検査を止めてくれ」。伝染します(any に触れた結果も any になる)。
  第 8 章で見たとおり、**入れた瞬間にその先の守りが全部消える** 非常口です
- **`unknown`** = 「何かは分からないが、**確かめるまで使わせない**」。安全な不明

**「型が分からない」場面では常に `unknown` を選び、narrowing で絞ってから使う。**
`any` が正当化されるのは、移行中の古いコードとの境界くらいです。

もう一つの危険な呪文が **型アサーション `as`** です:

```typescript
const q = JSON.parse(text) as Quest;   // 「Quest だということにしてくれ」
```

`as` は変換ではなく **検査の免除** です。実際の中身が Quest でなくても黙って通ります。
「私は TypeScript より正しい」と宣言する呪文なので、使うたびに一度疑ってください。
では `JSON.parse` のような「本当に中身が分からない」データはどうすべきか——それが
次章の主題です。

## as const と satisfies — 値を型で引き締める

```typescript
// as const: 値を「そのリテラルのまま・readonly」で固定する
const RANKS = ["bronze", "silver", "gold"] as const;
type Rank = (typeof RANKS)[number];   // "bronze" | "silver" | "gold"
//   ↑ 値の配列から union 型を生成する頻出イディオム(値と型の二重管理が消える)

// satisfies: 「この型を満たすことだけ検査し、推論結果は細かいまま保つ」
const rewardTable = {
  bronze: 30,
  silver: 80,
  gold: 200,
} satisfies Record<Rank, number>;     // キーの過不足を検査。typo や書き忘れを検出
```

## ⚔️ 完成コード: `guild/src/quests.ts` に組み込む

```typescript
// 第 6 章の quests.ts を魔導書で強化

export type QuestPatch = Partial<Omit<Quest, "id" | "status">>;
//     「id と status 以外を、任意の組み合わせで修正できる」— 型の合成技

export function updateQuest(id: number, patch: QuestPatch): Quest | undefined {
  const q = quests.find((q) => q.id === id);
  if (!q) return undefined;
  Object.assign(q, patch);          // patch のプロパティを q に上書き
  return q;
}

export function summarize(): Pick<Quest, "id" | "title">[] {
  return quests.map((q) => ({ id: q.id, title: q.title }));
}
```

```typescript
// main.ts で
updateQuest(1, { reward: 45 });                  // ✅ 報酬の改定
updateQuest(1, { title: "薬草採取(急ぎ)" });   // ✅ タイトル修正
updateQuest(1, { id: 999 });                     // ❌ id の改竄は型で禁止
updateQuest(1, { status: { state: "open" } });   // ❌ 状態は第 5 章の関数経由でしか変えられない
```

**「変えてよいもの」を型が定義している** ことに注目してください。第 5 章では
「ありえない状態」を、今日は「ありえない操作」を、型で消しました。

## 📝 今日の受付業務(演習)

1. `MyRequired<T>`(`Partial` の逆: 全プロパティを必須にする)を mapped type で自作してください(ヒント: `?` を外す記法は `-?`)。
2. `type Adventurer = { name: string; rank: Rank; wallet: number }` から、`Pick` / `Omit` / `Readonly` でそれぞれ型を作り、エディタで中身(マウスホバー)を確認してください。
3. `const DIFFICULTIES = ["easy", "normal", "hard"] as const;` から `Difficulty` union 型を導出し、第 5 章の演習 4 を「値と型の一元管理」に書き直してください。
4. `JSON.parse('{"id":1}') as Quest` で作った偽クエストの `title.toUpperCase()` を呼ぶとどうなりますか?コンパイルと実行の結果を分けて記録してください——次章の事件の予告編です。

---

次章、その事件が起きます。外部から届いた依頼書ファイルを `as Quest` で受け入れた結果、
夜中にギルドのシステムが停止。**型は実行時には存在しない** ——第 1 章から予告し続けた
最重要の帰結と、その防衛術です。 → [第14章 門番の検問](14_runtime_validation.md)
