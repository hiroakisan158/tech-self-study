# 第11章 技を巻物にする — カスタムフックとフックのルール

## 🎭 今日のお話

劇場のあちこちで同じ仕掛けが要ることに気づきました。ロビーの時計も、舞台袖の時計も、
楽屋の時計も「1 秒ごとに時を刻む」。窓口のメモも、監督のノートも「ブラウザを閉じても
内容が残ってほしい」。

state と effect の **組み合わせ方** に再利用したいパターンが生まれています。関数の
ロジックなら[関数に切り出す](../../04-typescript-fable-101/chapters/04_functions.md)のが常道——
実はフックも同じです。**フックを使う関数を自作すれば、それがカスタムフック** です。
コピペしてきた技を、巻物に書き写して道具箱に収める日です。

## カスタムフック — 「フックを使う関数」を自作する

第 9 章のカウントダウンから「時を刻む」部分だけを抜き出します:

```tsx
// src/hooks/useCountdown.ts — ただの関数。ただしフックを使うので名前は use で始める
import { useState, useEffect } from "react";

export function useCountdown(initialSeconds: number) {
  const [secondsLeft, setSecondsLeft] = useState(initialSeconds);

  useEffect(() => {
    const timer = setInterval(() => {
      setSecondsLeft((s) => Math.max(0, s - 1));
    }, 1000);
    return () => clearInterval(timer);
  }, []);

  const isTimeUp = secondsLeft === 0;
  return { secondsLeft, isTimeUp };   // 好きな形で返せる(オブジェクトでもタプルでも)
}
```

使う側は、内部の仕掛けを一切知らずに済みます:

```tsx
function LobbyClock() {
  const { secondsLeft, isTimeUp } = useCountdown(300);
  return <p>⏰ {isTimeUp ? "開演!" : `あと ${secondsLeft} 秒`}</p>;
}

function BackstageClock() {
  const { secondsLeft } = useCountdown(300);   // 別の呼び出し = 別の記憶箱
  return <small>袖時計: {secondsLeft}</small>;
}
```

押さえるべき性質は 2 つです:

- **カスタムフックは「ロジック」を共有し、「state」は共有しない。**
  呼び出しごとに独立した記憶箱が作られます(上の 2 つの時計は別々に時を刻みます)。
  state そのものを共有したいなら[リフトアップ](08_lifting_state.md)か次章の Context です
- **名前は必ず `use` で始めます。** ただの慣習ではなく、ESLint がこの名前を目印に
  「フックのルール」(後述)を検査するための契約です

> 💡 カスタムフックの中身は「useState と useEffect の組み合わせ」だけではありません。
> 他のカスタムフックを呼ぶこともできます。フックの合成——
> [部品の合成](02_props.md)と同じ思想が、ロジックの層でも成立します。

## もう一本の巻物 — useLocalStorage

「ブラウザを閉じても残る state」。`localStorage`(ブラウザ内の永続保存領域)との
同期をフックに封じます:

```tsx
// src/hooks/useLocalStorage.ts
import { useState, useEffect } from "react";

export function useLocalStorage(key: string, initialValue: string) {
  const [value, setValue] = useState(() => {
    // 初期値の「遅延初期化」: 初回レンダーで 1 回だけ実行される関数を渡せる
    return localStorage.getItem(key) ?? initialValue;
  });

  useEffect(() => {
    localStorage.setItem(key, value);   // value が変わるたび保存
  }, [key, value]);

  return [value, setValue] as const;    // useState と同じ [値, 更新関数] の顔にする
}
```

```tsx
function DirectorNote() {
  const [note, setNote] = useLocalStorage("director-note", "");
  return <textarea value={note} onChange={(e) => setNote(e.target.value)} />;
}
```

`useState` と同じ顔(タプル)で返しているので、使い心地は「保存機能付き useState」。
**インターフェースを揃えて中身だけ差し替え可能にする**——
[構造的型付け](../../04-typescript-fable-101/chapters/03_objects_arrays.md)の精神です。
`as const` は[タプルとして推論させるための一手](../../04-typescript-fable-101/chapters/13_advanced_types.md)です
(これがないと `(string | ...)[]` という緩い配列型になってしまいます)。

## フックのルール — なぜ if の中で呼べないのか

第 5 章から棚上げしてきた謎を解きます。フックには絶対のルールが 2 つあります:

1. **コンポーネント(またはカスタムフック)の最上位でのみ呼ぶ。**
   if・for・ネストした関数・early return の後では呼ばない
2. **React の関数の中でのみ呼ぶ。** 普通の関数やクラスからは呼ばない

```tsx
function BadActor({ isVip }: { isVip: boolean }) {
  if (isVip) {
    const [perks, setPerks] = useState<string[]>([]);   // ❌ 条件の中で useState
  }
  const [name, setName] = useState("");
  ...
}
```

> ⚙️ **舞台裏の真実 — 記憶箱は「呼び出し順」で照合されている**
>
> 前章で「state は木の中の位置に住む」と学びました。では、1 つのコンポーネントが
> 借りた **複数の** 記憶箱を、React はどう見分けているのでしょう?
>
> 答えは拍子抜けするほど単純です: **呼ばれた順番** です。React は各コンポーネント位置に
> 記憶箱のリストを持ち、「1 回目の useState はこの箱、2 回目はこの箱……」と
> **順番だけで** 対応づけています。変数名もキーもありません。
>
> だから順番が狂うと大惨事です。上の `BadActor` で `isVip` が true → false に変わると、
> 前回 1 番目だった `perks` の箱が、今回 1 番目に呼ばれた `name` に渡されます——
> **他人の記憶を掴む** のです。「最上位でのみ呼ぶ」ルールは、この照合を守るための
> 「毎回、必ず、同じ順番で全記憶箱を借りに来ること」という契約です。
>
> なぜこんな設計に?名前やキーを毎回書かせる案もあり得ましたが、React チームは
> 「書き味の軽さ」を優先し、順番による暗黙の照合 + Lint による機械的検査を選びました。
> 割り切りの設計です——ルールの理由さえ知っていれば、怖いものではありません。

条件つきの挙動が欲しいときは、**フックは無条件で呼び、条件は中で処理する** のが作法です:

```tsx
const [perks, setPerks] = useState<string[]>([]);   // 箱は必ず借りる
const visiblePerks = isVip ? perks : [];             // 使うかどうかを条件にする
```

> 📜 **歴史の背景 — フックはコミュニティの「共有の器」になった**
>
> Hooks 導入(2019)の隠れた最大の成果は、**ロジックの共有単位** が生まれたことでした。
> クラス時代、ロジックの再利用は HOC や render props という複雑なパターンに
> 頼っていました。カスタムフックは「ただの関数」なので、書くのも配るのも
> [npm に公開する](../../04-typescript-fable-101/chapters/15_build_ecosystem.md)のも簡単です。
>
> 結果、コミュニティには `useDebounce`、`useMediaQuery`、`useForm`(react-hook-form)、
> `useQuery`(TanStack Query、第 14 章で言及)など無数の巻物が流通しています。
> 「まず誰かの巻物を探す、なければ書く」が現代 React の日常です。

## ⚔️ 完成コード: 巻物 2 本と時計たち

```tsx
// src/App.tsx — Reactive Theater 11 日目: 道具箱の整備

import { useCountdown } from "./hooks/useCountdown";
import { useLocalStorage } from "./hooks/useLocalStorage";

function LobbyClock() {
  const { secondsLeft, isTimeUp } = useCountdown(600);
  const mm = Math.floor(secondsLeft / 60);
  const ss = String(secondsLeft % 60).padStart(2, "0");
  return (
    <section>
      <h2>🪧 ロビー</h2>
      <p style={{ fontSize: "1.5rem" }}>{isTimeUp ? "🔔 開演!" : `開演まで ${mm}:${ss}`}</p>
    </section>
  );
}

function DirectorNote() {
  const [note, setNote] = useLocalStorage("director-note", "");
  return (
    <section>
      <h2>📓 監督ノート(自動保存)</h2>
      <textarea
        value={note}
        onChange={(e) => setNote(e.target.value)}
        rows={4}
        cols={40}
        placeholder="今日の反省点…(リロードしても消えません)"
      />
    </section>
  );
}

function App() {
  return (
    <main>
      <h1>🎭 Reactive Theater</h1>
      <LobbyClock />
      <DirectorNote />
    </main>
  );
}

export default App;
```

監督ノートに何か書いてから **ページをリロード** してください。内容が残っています。
「state と effect の合わせ技」が 1 行(`useLocalStorage(...)`)に畳まれた手応えを
確かめてください。

## 📝 今日の舞台稽古(演習)

1. `useCountdown` に `reset()` 関数を追加して返し、「もう一度最初から」ボタンを作ってください。
2. `useToggle(initial: boolean)` を自作してください: `const [isOpen, toggle] = useToggle(false)` で使える、切り替え専用の巻物です。
3. `useDocumentTitle(title: string)` を自作し、第 9 章の `StageTitle` を 1 行に置き換えてください。
4. わざと if の中で `useState` を呼び、ESLint(または実行時)のエラーメッセージを観察してください。そのメッセージを「記憶箱の照合」の言葉で翻訳してみましょう。

---

次章、劇場全体に関わる設定——照明(ダークモード)——を導入します。props で全部品に
バケツリレーする苦行を回避する道具、**Context** の出番です。ただし便利さには
代償もあります。 → [第12章 劇場全体の照明](12_context.md)
