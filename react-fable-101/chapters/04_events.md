# 第4章 客席からの拍手 — イベント処理

## 🎭 今日のお話

ここまでの劇場は、いわば **無観客公演** でした。画面は表示するだけで、観客(ユーザー)が
何をしても反応しません。今日はついに客席と舞台がつながります——拍手ボタン、
アンコールの声、完売演目の問い合わせ。

React のイベント処理は 1 行で書けます。しかしその 1 行には、
[TypeScript 第 4 章「関数は値である」](../../typescript-fable-101/chapters/04_functions.md)と
[第 11 章「イベントループ」](../../typescript-fable-101/chapters/11_event_loop.md)の知識が
まるごと流れ込んでいます。

## イベントハンドラ — 「呼ぶ」のではなく「渡す」

```tsx
function ApplauseButton() {
  function handleClick() {
    alert("👏 拍手が送られました!");
  }

  return <button onClick={handleClick}>拍手を送る</button>;
  //             ~~~~~~~~~~~~~~~~~~~~ 関数「そのもの」を渡す
}
```

最重要ポイントは `onClick={handleClick}` に **括弧がない** ことです。

```tsx
<button onClick={handleClick}>    // ✅ 関数を渡す。「クリックされたら呼んでね」
<button onClick={handleClick()}>  // ❌ 関数をいま呼ぶ。描画のたびに alert が鳴る
```

`handleClick()` と書くと、それは「レンダリング中に関数を実行し、**その戻り値**(ここでは
`undefined`)を渡す」という意味になってしまいます。第 1 章で学んだとおり JSX の `{}` は
ただの式だからです。

- **渡すのは関数という値**(コールバック)。押された「後で」React が呼びます
- これは [TS 第 11 章の `setTimeout(fn, ...)`](../../typescript-fable-101/chapters/11_event_loop.md) と
  完全に同じ構図です。ブラウザのイベントループが「クリック」をタスクとして受け取り、
  あなたが預けた関数を呼ぶ——React はその配線を整えているだけです

引数を渡したいときは、**アロー関数で包みます**:

```tsx
function ShowRow({ show }: { show: Show }) {
  return (
    <li>
      {show.title}
      <button onClick={() => reserve(show.id)}>予約</button>
      {/*             ~~~~~~~~~~~~~~~~~~~~~~~ 「押されたら reserve(id) を呼ぶ関数」を渡す */}
    </li>
  );
}
```

`reserve(show.id)` と直接書けば即実行、`() =>` で包めば「後で実行する包み」になる——
この使い分けが腑に落ちれば、今日の峠は越えています。

> 💡 **命名の慣習**: ハンドラ関数は `handle○○`(handleClick, handleSubmit)、
> それを受け取る props 名は `on○○`(onClick, onReserve)が業界標準です。
> 「on = 出来事の名前、handle = それに対処する側」と覚えてください。

## イベントオブジェクト — 出来事の詳細報告

ハンドラには **イベントオブジェクト** が渡されます。「何がどこで起きたか」の報告書です。

```tsx
function GuestBook() {
  function handleChange(e: React.ChangeEvent<HTMLInputElement>) {
    console.log(`入力中: ${e.target.value}`);   // 入力欄のいまの値
  }

  function handleSubmit(e: React.FormEvent<HTMLFormElement>) {
    e.preventDefault();    // ブラウザ既定の動作(ページ再読み込み)を止める
    console.log("芳名帳に記帳されました");
  }

  return (
    <form onSubmit={handleSubmit}>
      <input onChange={handleChange} placeholder="お名前" />
      <button type="submit">記帳する</button>
    </form>
  );
}
```

- 型は `React.ChangeEvent<要素の型>` のように書きます。**覚える必要はありません**——
  `onChange={(e) => ...}` とインラインで書けば、[文脈から推論](../../typescript-fable-101/chapters/04_functions.md)
  されます。関数を外に切り出すときだけ明示が要る、と覚えれば十分です
- `e.preventDefault()` は頻出です。HTML のフォームは送信時に **ページ全体を再読み込みする**
  のが 1990 年代からの既定動作で、シングルページアプリではそれを止めて自分で処理します

> 📜 **歴史の背景 — onclick="..." から onClick={fn} へ**
>
> HTML にも `onclick` 属性は大昔からありますが、中身は **文字列** でした
> (`onclick="reserve(1)"`)。文字列なのでスコープは滅茶苦茶、typo は実行時まで不明、
> 渡せる情報も限定的——「HTML と JS を分離せよ」という当時の教義は、この仕組みの
> ひどさへの防衛策でもありました。
>
> React の `onClick={handleClick}` は見た目こそ先祖返りですが、中身は別物です。
> 渡すのは文字列ではなく **スコープとクロージャを持った本物の関数**
> ([TS 第 9 章](../../typescript-fable-101/chapters/09_array_methods.md))。上の `() => reserve(show.id)` が
> `show` を覚えていられるのはクロージャのおかげです。「見た目とロジックを機能単位で
> まとめる」(第 1 章)が健全に実現できるのは、関数が第一級の値である言語の上だからです。

> ⚙️ **舞台裏の真実 — 合成イベントとイベント委譲**
>
> `e` は生のブラウザイベントを React が薄く包んだ **SyntheticEvent(合成イベント)** です。
> 歴史的にはブラウザ間の仕様差を吸収するための層でした(IE との戦いの遺構)。
> また React は、各ボタンに個別にリスナーを付けるのではなく、アプリのルート要素で
> **一括受信して該当コンポーネントに配達する**(イベント委譲)方式をとっています。
> 1 万行のリストにハンドラを書いてもリスナーは実質 1 つ——普段意識する必要はありませんが、
> 「React がブラウザとあなたの間に立っている」ことを知っておくと、生の DOM 操作と
> 混ぜたときの不思議な挙動に説明がつきます。

## 押しても変わらない?— 次章への崖

拍手の **回数** を数えたくなるのが人情です。素直に書いてみます:

```tsx
function ApplauseCounter() {
  let count = 0;   // 拍手の回数……のつもり

  function handleClick() {
    count = count + 1;
    console.log(`拍手 ${count} 回目`);   // コンソールでは増えている!
  }

  return <button onClick={handleClick}>👏 {count}</button>;
  //                                       ~~~~~~~ しかし画面は 0 のまま!
}
```

押すたびにコンソールの数字は増えるのに、**画面は 0 から動きません**。

理由はもう説明できるはずです。`UI = f(state)`——画面が変わるのは **React が関数を
呼び直したとき** だけです。ローカル変数をいくら書き換えても、React に「再上演してくれ」
と伝わりません。しかも仮に呼び直されたら、`let count = 0` の行で **記憶はリセット** されます。

必要なのは「**再上演をまたいで生き残り、変更したら再上演を予約してくれる記憶**」。
それが次章の `useState` です。

## ⚔️ 完成コード: `src/App.tsx`

```tsx
// Reactive Theater — 4 日目: 客席との交信

interface Show {
  id: number;
  title: string;
  startTime: string;
  soldOut: boolean;
}

const shows: Show[] = [
  { id: 1, title: "ハムレット", startTime: "19:00", soldOut: false },
  { id: 2, title: "真夏の夜の夢", startTime: "14:00", soldOut: true },
  { id: 3, title: "マクベス", startTime: "18:00", soldOut: false },
];

function ShowRow({ show, onReserve }: { show: Show; onReserve: (id: number) => void }) {
  //                     ~~~~~~~~~~ 「何が起きたか」を親に知らせるコールバック props
  return (
    <li>
      {show.title} — 開演 {show.startTime}{" "}
      {show.soldOut ? (
        <button onClick={() => alert("キャンセル待ちを受け付けました")}>
          キャンセル待ち
        </button>
      ) : (
        <button onClick={() => onReserve(show.id)}>予約する</button>
      )}
    </li>
  );
}

function App() {
  function handleReserve(id: number) {
    const show = shows.find((s) => s.id === id);
    alert(`🎫 「${show?.title ?? "?"}」の予約を受け付けました(仮)`);
  }

  return (
    <main>
      <h1>🎭 Reactive Theater</h1>
      <ul>
        {shows.map((show) => (
          <ShowRow key={show.id} show={show} onReserve={handleReserve} />
        ))}
      </ul>
    </main>
  );
}

export default App;
```

構図に注目してください。**データは props で下りていき(show)、出来事はコールバックで
上っていく(onReserve)**。子は「予約ボタンが押された」と報告するだけで、予約処理の
中身は知りません。この「props down, events up」が React アプリの基本の血流です。

## 📝 今日の舞台稽古(演習)

1. `onClick={handleClick}` を `onClick={handleClick()}` に書き換えて挙動を観察してください(画面表示のたびに何が起きますか?)。
2. 「アンコール」ボタンを追加し、押すたびにコンソールへ `📣 アンコール!` を出してください。`onClick={() => console.log(...)}` のインライン記法で。
3. `ShowRow` の `onReserve` を渡し忘れるとどうなるか確認してください(TypeScript の props 検査がここでも働きます)。
4. 崖の実験: 上の `ApplauseCounter`(壊れている版)を実際に作り、「コンソールは増えるのに画面が動かない」を自分の目で確認してください。次章の主役を迎える準備です。

---

次章、React の心臓部 **state** に到達します。役者に「記憶」を持たせ、
拍手カウンタを本当に動かします。React 学習の最大の山場、じっくり登りましょう。
→ [第5章 役者の記憶](05_state.md)
