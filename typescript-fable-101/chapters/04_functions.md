# 第4章 受付係を増やす — 関数

## 🍺 今日のお話

依頼が 1 日 20 件を超え、あなた一人では回らなくなりました。「依頼を掲示する」「報酬を
計算する」「掲示板を読み上げる」——同じ手順を毎回手作業でやるのはもう限界です。

手順に名前を付けて何度でも呼び出せるようにする仕組み、それが **関数** です。
今日は「手順書を書いて、係に任せる」日です。

## 関数の 3 つの書き方

JavaScript には関数の書き方が(歴史的経緯で)3 つあります。まず全部見てしまいましょう。

```typescript
// ① 関数宣言(function declaration)
function calcReward(base: number, taxRate: number): number {
  return base * (1 + taxRate);
}

// ② 関数式(function expression)— 関数を値として変数に代入
const calcReward2 = function (base: number, taxRate: number): number {
  return base * (1 + taxRate);
};

// ③ アロー関数(arrow function)— ES2015 で追加された現代の主流
const calcReward3 = (base: number, taxRate: number): number => {
  return base * (1 + taxRate);
};

// ③' アロー関数の省略形: 本体が式 1 つだけなら {} と return を省ける
const calcReward4 = (base: number, taxRate: number): number => base * (1 + taxRate);
```

> 📜 **歴史の背景 — なぜ 3 つもあるのか**
>
> ①② は 1995 年からある書き方です。③ のアロー関数は 2015 年に追加されました。
> 理由は 2 つ——(1) コールバック(後述)を短く書きたい、(2) 旧来の `function` が持つ
> `this` の混乱(第 10 章のお楽しみ)を避けたい、です。ここでも JavaScript は
> 「古い書き方を削除せず、新しい正解を横に足す」進化をしています。
>
> **この教材の方針**: トップレベルの「名前のある手順」は ① の `function` 宣言、
> その場で使い捨てる小さな関数(コールバック)は ③ のアロー関数で書きます。
> 実務ではすべてアロー関数で書く流派もあり、どちらも間違いではありません。

## 引数と戻り値の型

```typescript
function formatQuest(title: string, reward: number, done: boolean = false): string {
  //                 ~~~~~~~~~~~~~ 引数の型         ~~~~~~~~~~~~~ デフォルト値  ~~~~~~ 戻り値の型
  const mark = done ? "✅" : "🆕";
  return `${mark} ${title} — ${reward}G`;
}

formatQuest("薬草採取", 30);          // ✅ done は省略可(false になる)
formatQuest("薬草採取");              // ❌ エラー: reward がない
formatQuest(30, "薬草採取");          // ❌ エラー: 型の順序が違う(引数の渡し間違いを検出!)
```

- **引数の型注釈は必須**です(書かないと `any` になり、TypeScript の意味がなくなります)
- **戻り値の型は推論可能**なので省略もできますが、「この関数は何を返す約束か」を
  先に宣言しておくと、実装ミスを関数の中で検出できます。学習中は書くのがおすすめです
- 何も返さない関数の戻り値型は `void` と書きます

```typescript
function announce(message: string): void {
  console.log(`📢 ${message}`);
}
```

## 可変長引数(rest parameters)

「何件でも受け取る」関数は `...` で書けます。

```typescript
function totalReward(...rewards: number[]): number {
  let sum = 0;
  for (const r of rewards) sum += r;
  return sum;
}

totalReward(30, 80, 15);      // 125
totalReward();                // 0
```

第 3 章のスプレッド構文と同じ `...` 記号ですが、役割は逆向きです:
**関数定義側の `...` は「集める」、呼び出し側の `...` は「ばらまく」**。

```typescript
const rewards = [30, 80, 15];
totalReward(...rewards);      // 配列をばらまいて渡す → 125
```

## 関数は「値」である — JavaScript の魂

ここが今日いちばん大事な話です。JavaScript では **関数そのものが値** です。
数値を変数に入れたり引数に渡したりできるのと同じように、**関数を変数に入れ、
引数として渡し、戻り値として返せます**。

```typescript
// 関数を受け取る関数(高階関数)
function repeatTask(times: number, task: (n: number) => void): void {
  //                               ~~~~~~~~~~~~~~~~~~~~~ 「number を受け取り何も返さない関数」型
  for (let i = 1; i <= times; i++) {
    task(i);
  }
}

// 関数を「引数として」渡す。渡される側の関数をコールバックと呼ぶ
repeatTask(3, (n) => console.log(`${n} 回目の見回り完了`));
```

`(n: number) => void` は **関数の型** です。「引数と戻り値がこの形の関数なら何でも渡せる」
——オブジェクトの形を interface で縛ったのと同じ発想を、関数にも適用できるわけです。

> 📜 **歴史の背景 — Scheme の血**
>
> ブレンダン・アイクはもともと「ブラウザに関数型言語 Scheme を載せる」ために Netscape に
> 採用されました。経営判断で「見た目は Java 風に」と命じられましたが、**「関数は第一級の
> 値である」という Scheme の魂** はこっそり残されました。
>
> この設計が、のちの JavaScript の全てを決めます。ボタンがクリックされたら関数を呼ぶ
> (イベントリスナー)、データが届いたら関数を呼ぶ(コールバック、第 11 章)、配列の
> 各要素に関数を適用する(`map`/`filter`、第 9 章)——現代のフロントエンドも Node.js も、
> 「関数を値として渡す」文化の上に建っています。React のコンポーネントもただの関数です。

さっそく実用例を。配列の `sort` は「並べ方を決める関数」を受け取ります。

```typescript
const rewards = [30, 80, 15];
rewards.sort((a, b) => a - b);   // 昇順: [15, 30, 80]
rewards.sort((a, b) => b - a);   // 降順: [80, 30, 15]
```

💡 コールバックの引数 `a`, `b` に型注釈がないのにエラーにならないことに気づきましたか?
`rewards` が `number[]` だと分かっているので、TypeScript が **文脈から型を推論**
しています(contextual typing)。コールバックの型注釈は普通、省略で OK です。

## スコープ — 名前の見える範囲

```typescript
const guildName = "Typed Tavern";   // トップレベル: どこからでも見える

function greet(): void {
  const visitor = "旅の剣士";        // 関数の中: この関数の中だけ
  if (true) {
    const secret = "合言葉";         // ブロックの中: この {} の中だけ
  }
  // console.log(secret);           // ❌ エラー: ブロックの外からは見えない
  console.log(`${guildName} へようこそ、${visitor}`);  // ✅ 外側の名前は見える
}
```

`let`/`const` は `{}` 単位のブロックスコープです(第 1 章で触れた `var` はこれを守らない
のでした)。「内側から外側は見える、外側から内側は見えない」——この非対称性が、
第 9 章のクロージャという強力な魔法の土台になります。

## ⚔️ 完成コード: `guild/day4.ts`

```typescript
// Typed Tavern — 4 日目: 受付業務の自動化

interface Quest {
  id: number;
  title: string;
  reward: number;
  done: boolean;
}

const quests: Quest[] = [
  { id: 1, title: "薬草採取", reward: 30, done: false },
  { id: 2, title: "ゴブリン退治", reward: 80, done: false },
  { id: 3, title: "酒場の看板修理", reward: 15, done: true },
];

let nextId = 4;

function postQuest(title: string, reward: number): Quest {
  const quest: Quest = { id: nextId, title, reward, done: false };
  //                     💡 title: title は title と省略できる(ショートハンド)
  nextId += 1;
  quests.push(quest);
  return quest;
}

function formatQuest(q: Quest): string {
  const mark = q.done ? "✅" : "🆕";
  return `${mark} [${q.id}] ${q.title} — ${q.reward}G`;
}

function showBoard(title: string): void {
  console.log(`📌 ===== ${title} =====`);
  for (const q of quests) {
    console.log(formatQuest(q));
  }
}

function totalOpenReward(): number {
  let sum = 0;
  for (const q of quests) {
    if (!q.done) sum += q.reward;
  }
  return sum;
}

showBoard("開店時の掲示板");
postQuest("迷子の猫さがし", 20);
postQuest("古文書の解読", 120);
showBoard("依頼追加後の掲示板");
console.log(`\n受付中の依頼の報酬総額: ${totalOpenReward()}G`);
```

```bash
npx tsx guild/day4.ts
```

## 📝 今日の受付業務(演習)

1. `completeQuest(id: number): boolean` を書いてください。該当 ID の依頼を `done: true` にして `true` を返し、見つからなければ `false` を返します。
2. `showBoard` に「並べ方の関数」を渡せるようにしてください: `showBoard(title, (a, b) => b.reward - a.reward)` で報酬の高い順に表示できるように(ヒント: 引数の型は `(a: Quest, b: Quest) => number`、省略可にするなら `?` を)。
3. `formatQuest` の戻り値型注釈を `number` に書き換えると、エラーは **関数の中と呼び出し側のどちら** に出るでしょう?予想してから試してください。
4. アロー関数の省略形だけを使って `const double = ...` (数値を 2 倍する関数)を 1 行で書いてください。

---

次章、依頼に「受付中 → 冒険者が受注 → 完了」という **状態** が生まれます。
「受注されていない依頼に冒険者名を書く」ような矛盾を、型そのもので禁止できたら…?
TypeScript の華、union 型の登場です。 → [第5章 依頼の振り分け](05_unions.md)
