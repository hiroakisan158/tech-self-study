# 第7章 小道具は複製して替える — オブジェクト・配列の state と不変性

## 🎭 今日のお話

チケット窓口の予約を **予約台帳** に溜めることになりました。台帳は配列です。
新しい予約が来たら `push`……と手が動いたあなた、今日はその手を止める日です。

小道具係の鉄則をお教えします。**舞台に出した小道具は、直接いじらない。
差し替えたければ、複製を作って丸ごと取り替える。** なぜそんな回りくどいことを?
——理由は React の変更検知の仕組みそのものにあります。

## 事件 — push したのに画面が増えない

```tsx
function ReservationBook() {
  const [reservations, setReservations] = useState<string[]>([]);

  function handleAdd(name: string) {
    reservations.push(name);          // ❌ 台帳に直接書き込み
    setReservations(reservations);    // 同じ配列を渡して再上演を予約……したつもり
  }
  ...
}
```

`push` は実行されています。`console.log` すれば中身も増えています。
**しかし画面は更新されません。** 第 4 章の「変わらない事件」の再来ですが、今回は
`setReservations` を呼んでいるのに、です。

> ⚙️ **舞台裏の真実 — React は「中身」を見ない。「別物かどうか」だけを見る**
>
> `setReservations(x)` を受け取った React は、「前の state と `x` は違う値か?」を
> 確認してから再上演を予約します。このとき配列の中身を 1 件ずつ調べる……ことは
> **しません**。やるのは `Object.is(前の値, x)` ——実質、
> [**同じ参照かどうか**](../../typescript-fable-101/chapters/03_objects_arrays.md)の比較 1 回だけです。
>
> `push` しても配列は同一の実体のまま([「ラベルの貼り替え」は起きていない](../../typescript-fable-101/chapters/03_objects_arrays.md))。
> React の目には「前と同じ値が来た = 何も変わっていない = 再上演不要」と映ります。
>
> なぜ中身を調べないのか?台帳が 1 万件なら、全件比較は毎回 1 万回の照合になります。
> 参照比較なら **件数によらず一瞬**。「変更したら必ず新しい実体を作る」という規律を
> プログラマ側が守る代わりに、React は世界最速の変更検知を手に入れる——
> これが不変性(イミュータビリティ)という取引の正体です。
> [TS 第 9 章で「React では前提」と予告した](../../typescript-fable-101/chapters/09_array_methods.md)理由が、これです。

## 正解 — 複製して差し替える

[スプレッド構文と非破壊メソッド](../../typescript-fable-101/chapters/09_array_methods.md)の出番です。
**配列 state の三大操作** をイディオムとして手に馴染ませてください:

```tsx
// 追加: 複製 + 新要素
setReservations([...reservations, newName]);

// 削除: filter は新しい配列を返す(非破壊)
setReservations(reservations.filter((r) => r.id !== targetId));

// 更新: map で「該当だけ差し替えた」新しい配列を作る
setReservations(
  reservations.map((r) =>
    r.id === targetId ? { ...r, tickets: r.tickets + 1 } : r
  )
);
```

オブジェクト state も同じ発想です:

```tsx
const [settings, setSettings] = useState({ volume: 5, curtain: "red", lights: true });

setSettings({ ...settings, volume: 8 });   // ✅ 複製しつつ volume だけ上書き
settings.volume = 8;                        // ❌ 直接変更。React は気づかない
```

`{ ...r, tickets: r.tickets + 1 }` の「複製しつつ一部だけ変更」は、
[TS 第 3 章で「React で毎日書くことになる」と予告した](../../typescript-fable-101/chapters/03_objects_arrays.md)
あのイディオムです。今日から本当に毎日書きます。

💡 **避けるべき破壊的メソッド → 非破壊の代替**:
`push`/`unshift` → スプレッド、`splice` → `filter`/`toSpliced`、
`sort` → `toSorted`、`reverse` → `toReversed`、直接代入 `arr[i] = x` → `map`。
[破壊/非破壊の見分け](../../typescript-fable-101/chapters/09_array_methods.md)が
React では死活問題になります。

## ネストした state — 深い複製は浅く考える

予約がオブジェクトの配列になると、更新はこう見えます:

```tsx
interface Reservation {
  id: number;
  name: string;
  tickets: number;
}

const [reservations, setReservations] = useState<Reservation[]>([]);

// 「id=2 の枚数を +1」— 外側の配列も、対象のオブジェクトも、両方新品にする
setReservations((prev) =>
  prev.map((r) => (r.id === 2 ? { ...r, tickets: r.tickets + 1 } : r))
);
```

- 変更経路上のもの(配列と、id=2 のオブジェクト)だけ複製し、
  **触らない要素は同じ参照を使い回して構いません**(map がそうしてくれています)。
  全世界の deep copy は不要です
- `setReservations((prev) => ...)` と[更新関数](05_state.md)で書くのは、
  連続更新でも安全にするためです。配列・オブジェクト state では特にこの形を推奨します

💡 state のネストが 3 段以上になって複製がつらいなら、それは **state の形の設計を
見直すサイン** です(フラットに持つ、ID で参照する)。それでも深い構造が必要な場面では、
「直接変更するように書くと複製を自動生成してくれる」**Immer** というライブラリが
定番です(第 13 章の `useReducer` と好相性)。

## 「導出は計算で」— 台帳から看板を導く

台帳(state)さえ正しければ、集計はすべて[レンダリング中の計算](05_state.md)で済みます:

```tsx
const totalTickets = reservations.reduce((sum, r) => sum + r.tickets, 0);
const isFull = totalTickets >= CAPACITY;
const vipList = reservations.filter((r) => r.tickets >= 4);
```

state は台帳 1 つ。看板・満席判定・VIP リストはぜんぶ `f(state)` ——
[TS 第 9 章の集計パイプライン](../../typescript-fable-101/chapters/09_array_methods.md)が
そのまま画面に接続されます。

## ⚔️ 完成コード: `src/App.tsx`

```tsx
// Reactive Theater — 7 日目: 予約台帳

import { useState } from "react";

interface Reservation {
  id: number;
  name: string;
  tickets: number;
}

const CAPACITY = 20;

function ReservationBook() {
  const [reservations, setReservations] = useState<Reservation[]>([]);
  const [name, setName] = useState("");
  const [nextId, setNextId] = useState(1);

  const totalTickets = reservations.reduce((sum, r) => sum + r.tickets, 0);
  const remaining = CAPACITY - totalTickets;

  function handleAdd(e: React.FormEvent<HTMLFormElement>) {
    e.preventDefault();
    setReservations((prev) => [...prev, { id: nextId, name, tickets: 1 }]);
    setNextId((id) => id + 1);
    setName("");
  }

  function handleAddTicket(id: number) {
    setReservations((prev) =>
      prev.map((r) => (r.id === id ? { ...r, tickets: r.tickets + 1 } : r))
    );
  }

  function handleCancel(id: number) {
    setReservations((prev) => prev.filter((r) => r.id !== id));
  }

  return (
    <section>
      <h2>📖 予約台帳(残席 {remaining})</h2>
      <form onSubmit={handleAdd}>
        <input
          value={name}
          onChange={(e) => setName(e.target.value)}
          placeholder="お客さまのお名前"
        />
        <button type="submit" disabled={name.trim() === "" || remaining <= 0}>
          予約を追加
        </button>
      </form>
      <ul>
        {reservations.map((r) => (
          <li key={r.id}>
            {r.name} さま — {r.tickets} 枚{" "}
            <button onClick={() => handleAddTicket(r.id)} disabled={remaining <= 0}>
              +1 枚
            </button>{" "}
            <button onClick={() => handleCancel(r.id)}>キャンセル</button>
          </li>
        ))}
      </ul>
      {remaining <= 0 && <p>🈵 満席となりました。キャンセル待ちのみ受付中です。</p>}
    </section>
  );
}

function App() {
  return (
    <main>
      <h1>🎭 Reactive Theater</h1>
      <ReservationBook />
    </main>
  );
}

export default App;
```

追加・+1・キャンセル・満席判定——**破壊的操作ゼロ** で台帳が完全に動いています。
`key={r.id}` が正しい ID である(添字ではない)ことも、削除機能があるリストでは
効いています(第 3 章の伏線回収です)。

## 📝 今日の舞台稽古(演習)

1. 冒頭の「push 事件」を実際に作って、画面が更新されないことを確認してください。その後 `[...prev, name]` に直して復旧を。
2. 「-1 枚」ボタンを追加してください。ただし 1 枚未満にはならないこと(0 枚になったら自動でキャンセル扱いにする、まで出来たら上級)。
3. `key={r.id}` を `key={index}` に変えてから、先頭の予約をキャンセルしてみてください。何か不穏なことが起きないか——各行に「+1 枚」済みの状態を作ってから試すと、key の重要性が体感できます。
4. 予約に `vip: boolean` を追加し、「VIP に昇格」ボタンを `map` + スプレッドで実装してください。VIP の予約は 🌟 付きで表示を。

---

次章、劇場に部品が増えてきました。「窓口で予約すると、ロビーの残席表示も変わる」——
**離れたコンポーネント同士が同じ state を見る** にはどうするか。React 設計の要、
「state を持ち上げる」を学びます。 → [第8章 舞台監督に預ける](08_lifting_state.md)
