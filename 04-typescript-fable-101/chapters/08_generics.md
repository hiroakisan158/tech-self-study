# 第8章 万能保管庫 — ジェネリクス

## 🍺 今日のお話

ギルドの地下に保管庫を作ることになりました。クエストの記録、冒険者の登録証、押収した
呪いのアイテム——**種類ごとに別々の棚** で保管したい。しかし棚のクラスを
`QuestVault`、`AdventurerVault`、`ItemVault`…と種類の数だけコピペで書くのは悪夢です。

「**棚の仕組みは 1 回だけ書き、何を入れる棚かは後から指定する**」——これを可能にするのが
**ジェネリクス** です。

## 問題: any の棚は型の守りを失う

まず、ジェネリクスなしでどう困るかを体験します。

```typescript
// 何でも入る棚(any 版)
class AnyVault {
  private items: any[] = [];
  add(item: any): void { this.items.push(item); }
  get(index: number): any { return this.items[index]; }
}

const vault = new AnyVault();
vault.add({ id: 1, title: "薬草採取" });
vault.add("金貨袋");                     // 違う種類も黙って入ってしまう

const q = vault.get(0);
console.log(q.titel);                    // typo なのにエラーにならない!(any は検査停止)
```

`any` は「型検査をやめてください」という宣言です。入れた瞬間の型情報が失われるので、
取り出したものが何なのか TypeScript はもう分かりません。第 1 章の演習で見た
「危険な型」の正体がこれです。

## 型引数 `<T>` — 型を「あとで決める」

```typescript
class Vault<T> {
  private items: T[] = [];

  add(item: T): void {
    this.items.push(item);
  }

  get(index: number): T | undefined {
    return this.items[index];
  }

  get size(): number {
    return this.items.length;
  }
}
```

`<T>` は **型の引数** です。関数の引数 `(x: number)` が「値を後から受け取る」のと同じように、
`<T>` は「**型を後から受け取る**」ための穴です(T は Type の頭文字の慣習。何でもいい)。

```typescript
import { type Quest } from "./quests.js";
import { Adventurer } from "./adventurers.js";

const questVault = new Vault<Quest>();        // T = Quest で棚を 1 つ作る
const roster = new Vault<Adventurer>();       // T = Adventurer で別の棚を作る

questVault.add({ id: 1, title: "薬草採取", reward: 30, status: { state: "open" } });
questVault.add(new Adventurer("カイ"));       // ❌ エラー!クエスト棚に冒険者は入らない

const q = questVault.get(0);                  // q の型は Quest | undefined
if (q) console.log(q.title);                  // 取り出しても型が保たれている。typo も検出される
```

**書くのは 1 回、守りは棚ごと。** これがジェネリクスの全てです。

💡 `import { type Quest }` の `type` は「これは型としてだけ使う(実行時のコードには
不要)」という印です。型消去の世界観が import 文にも顔を出しています。

## ジェネリック関数

クラスだけでなく関数にも型引数を付けられます。

```typescript
function firstItem<T>(items: T[]): T | undefined {
  return items[0];
}

const n = firstItem([10, 20, 30]);        // n の型は number | undefined
const s = firstItem(["a", "b"]);          // s の型は string | undefined
```

呼び出し時に `firstItem<number>(...)` と明示しなくても、**引数から T が推論されます**。
実はあなたはすでにジェネリクスの恩恵を受けています——第 4 章で `sort` のコールバックの
引数型が自動で分かったのは、`Array<T>` のメソッドだからです。

## 型引数に条件を付ける — `extends` 制約

「ID を持つものなら何でも」のような条件付きの棚を作りたいときは、`extends` で
**T に最低限の形を要求** します。

```typescript
interface HasId {
  id: number;
}

function findById<T extends HasId>(items: T[], id: number): T | undefined {
  return items.find((item) => item.id === id);   // T は必ず id を持つと保証されている
}

findById(questList, 2);      // ✅ Quest は id を持つ
findById(["a", "b"], 1);     // ❌ エラー: string は id を持たない
```

制約がないと、関数本体で `item.id` に触れることすらできません(T は「何でもありうる」
ので)。**制約は「T に要求する最低限のプロフィール」** です。

## 標準ライブラリはジェネリクスでできている

実は、日常的に使う組み込みの入れ物はぜんぶジェネリックです。

```typescript
const titles: Array<string> = [];            // string[] と同じもの(配列の正式名)
const stock: Map<string, number> = new Map(); // キーと値の型を 2 つ取る辞書
const visited: Set<number> = new Set();       // 重複なしの集合

stock.set("回復薬", 12);
stock.set("解毒剤", 3);
console.log(stock.get("回復薬"));   // 12(型は number | undefined)

visited.add(1);
visited.add(1);                     // 重複は無視される
console.log(visited.size);          // 1
```

💡 **Map とオブジェクトの使い分け**: 「決まった名前のプロパティを持つ 1 件の記録」は
オブジェクト(interface)、「実行時に増減するキー→値の対応表」は `Map` が適役です。
[Python の dict](../../02-python-fable-101/chapters/02_collections.md) の役割が、JS では
「構造はオブジェクト、辞書は Map」に分業されているイメージです。

第 12 章で登場する `Promise<T>`(「あとで T が届く約束」)もジェネリクスです。
ジェネリクスは特殊な上級機能ではなく、**エコシステム全体の共通語** です。

> 📜 **歴史の背景 — ジェネリクスと言語の性格、そしてアンダース・ヘルスバーグ**
>
> TypeScript は 2012 年、Microsoft の **アンダース・ヘルスバーグ** の設計で生まれました。
> 彼は Turbo Pascal、Delphi、C# を設計した「静的型付け言語の巨匠」で、C# に洗練された
> ジェネリクスを導入した張本人です。その血統ゆえ、TypeScript は **初期からジェネリクスを
> 中心に据えて** 設計されました。
>
> 対照的に [Go はジェネリクスを 13 年間拒み続けました](../../03-go-fable-101/chapters/11_generics.md)
> (「言語は単純であるべき」という思想のため)。同じ機能への態度に、言語の性格がこれほど
> 出るのは面白いところです。TypeScript の使命は「JavaScript の野生の柔軟なコードに型を
> 付けきる」ことなので、強力な型の道具立ては選択ではなく必然でした。その行き着いた先が
> 第 13 章の「型の魔導書」です。

## ⚔️ 完成コード: `guild/src/vault.ts`

```typescript
// Typed Tavern — 8 日目: 地下保管庫

interface HasId {
  id: number;
}

export class Vault<T extends HasId> {
  #items = new Map<number, T>();

  add(item: T): void {
    if (this.#items.has(item.id)) {
      console.log(`⚠️ ID ${item.id} は登録済みです`);
      return;
    }
    this.#items.set(item.id, item);
  }

  find(id: number): T | undefined {
    return this.#items.get(id);
  }

  remove(id: number): boolean {
    return this.#items.delete(id);
  }

  get all(): T[] {
    return [...this.#items.values()];   // Map の中身を配列にして返す
  }

  get size(): number {
    return this.#items.size;
  }
}
```

```typescript
// guild/src/main.ts での利用イメージ

import { Vault } from "./vault.js";
import { type Quest, postQuest } from "./quests.js";

interface Item {
  id: number;
  name: string;
  cursed: boolean;
}

const itemVault = new Vault<Item>();
itemVault.add({ id: 1, name: "呪いの仮面", cursed: true });
itemVault.add({ id: 2, name: "銀の短剣", cursed: false });
itemVault.add({ id: 1, name: "偽造ギルド証", cursed: true });  // ⚠️ ID 重複で拒否される

console.log(`🗄️ 保管庫: ${itemVault.size} 点`);
const found = itemVault.find(1);
console.log(found?.name ?? "該当なし");
```

## 📝 今日の受付業務(演習)

1. `Vault` に `has(id: number): boolean` と `clear(): void` を追加してください(`Map` のメソッドを調べて委譲するだけです)。
2. `lastItem<T>(items: T[]): T | undefined` をジェネリック関数で書き、`number[]` と `string[]` の両方で型がどう推論されるかエディタで確認してください。
3. `Vault<string>` を作ろうとするとどんなエラーが出ますか?エラー文の中に `HasId` 制約がどう現れるか観察してください。
4. `pluckIds<T extends HasId>(items: T[]): number[]`(全要素の id だけを配列で返す)を書いてください。次章で学ぶ `map` を予習して 1 行で書けたら見事です。

---

次章、月末がやってきました。帳簿の集計——「完了依頼だけ抜き出し、報酬を合計し、
冒険者ランキングを作る」。for ループの 10 行が、`filter` と `reduce` で 1 行になります。
→ [第9章 帳簿の集計](09_array_methods.md)
