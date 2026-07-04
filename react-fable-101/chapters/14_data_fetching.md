# 第14章 批評家の記事を取り寄せる — データ取得と非同期の設計

## 🎭 今日のお話

新聞社から「劇評をデータで提供できます」と連絡が来ました。ロビーに **劇評コーナー** を
作り、サーバーから記事を取り寄せて表示します。

これまでの劇場は自前のデータ(state)だけで回っていました。今日初めて
**ネットワークの向こう側**——遅延があり、失敗があり、
[型の保証がない世界](../../typescript-fable-101/chapters/14_runtime_validation.md)——と接続します。
TS 教材の後半で鍛えた async/await と門番の技術を、React の作法に載せ替える日です。

## 準備 — 劇評データを置く

外部 API の代わりに、プロジェクトの `public/` フォルダにファイルを置きます
(Vite は `public/` の中身をそのまま配信するので、`fetch("/reviews.json")` で取得できます)。

```json
// public/reviews.json
[
  { "id": 1, "critic": "劇評家アオヤマ", "stars": 5, "comment": "圧巻のハムレット。今年最高の舞台。" },
  { "id": 2, "critic": "夕刊シアター", "stars": 4, "comment": "装置転換の速さに唸った。" },
  { "id": 3, "critic": "観劇日和", "stars": 3, "comment": "熱演だが、幕間が長すぎる。" }
]
```

## 通信は 3 局面 — まず「状態の形」から設計する

コードより先に、画面の局面を数えます。通信には必ず 3 つの局面があります:
**読み込み中(loading)・成功(success)・失敗(error)**。

これを `useState` 3 連発(`isLoading`, `data`, `error`)で持つと、
「loading = false なのに data も error もない」「data と error が両方ある」といった
**ありえない組み合わせ** が型の上で作れてしまいます。
[第 5 章の教え](../../typescript-fable-101/chapters/05_unions.md)に従い、
判別可能 union で「ありえない局面」を消してから書き始めます:

```tsx
interface Review {
  id: number;
  critic: string;
  stars: number;
  comment: string;
}

type FetchState =
  | { status: "loading" }
  | { status: "success"; reviews: Review[] }
  | { status: "error"; message: string };
```

## effect でデータを取る — なぜ effect なのか

「劇評コーナーが **画面に表示されている** から取得する」——きっかけはユーザー操作では
なく表示そのもの。つまり[第 9 章の分類](09_effects.md)で **effect の管轄** です。

```tsx
import { useState, useEffect } from "react";

function ReviewCorner() {
  const [state, setState] = useState<FetchState>({ status: "loading" });

  useEffect(() => {
    let ignore = false;                    // 後述: 遅れて届いた過去の返事を無視する旗

    async function load() {
      try {
        const res = await fetch("/reviews.json");
        if (!res.ok) throw new Error(`HTTP ${res.status}`);
        const data: unknown = await res.json();          // 外から来たものは unknown(門番の掟)
        if (!ignore) setState({ status: "success", reviews: data as Review[] });  // 検証は後述で強化
      } catch (err) {
        if (!ignore) {
          setState({
            status: "error",
            message: err instanceof Error ? err.message : "不明なエラー",
          });
        }
      }
    }
    load();

    return () => { ignore = true; };        // クリーンアップで旗を立てる
  }, []);

  // 局面ごとの画面 — narrowing がそのまま UI の分岐になる
  switch (state.status) {
    case "loading":
      return <p>📰 劇評を取り寄せ中…</p>;
    case "error":
      return <p>⚠️ 劇評を取得できませんでした({state.message})</p>;
    case "success":
      return (
        <ul>
          {state.reviews.map((r) => (
            <li key={r.id}>
              {"⭐".repeat(r.stars)} {r.comment} — <em>{r.critic}</em>
            </li>
          ))}
        </ul>
      );
  }
}
```

`switch (state.status)` の各分岐で、TypeScript は `state.reviews` や `state.message` に
安全にアクセスさせてくれます。**通信の設計図(union)が、そのまま画面の分岐になる**——
TS 教材と React 教材の合流点です。

> ⚙️ **舞台裏の真実 — `ignore` 旗は競合状態(race condition)の防波堤**
>
> `useEffect(fn, [showId])` のように **対象を切り替えながら** 取得する場合を考えます。
> ハムレットの劇評を取得中に、ユーザーがマクベスに切り替えた——このとき
> [2 羽のフクロウが同時に飛んでいて](../../typescript-fable-101/chapters/11_event_loop.md)、
> **遅い方(ハムレット)が後から届くと、マクベスの画面にハムレットの劇評が表示されます**。
> ネットワークの返答順は保証されないからです。
>
> クリーンアップ(切り替え時に走る — [第 9 章](09_effects.md))で `ignore = true` を
> 立てておけば、旧世代の effect に届いた返事は `if (!ignore)` で捨てられます。
> [クロージャ](../../typescript-fable-101/chapters/09_array_methods.md)が世代ごとの `ignore` を
> 閉じ込めているので、旗は世代間で混線しません。より本格的には `AbortController` で
> 通信自体を中断します(演習 4)。
>
> ——と、正しく書くだけでこれだけの罠(競合・中断・エラー・再試行・キャッシュ)が
> あります。実務では **TanStack Query** などのライブラリや、Next.js のようなフレームワークの
> データ取得機構に任せるのが現代の定石です。ただしそれらの内部でこの章と同じことが
> 起きています。**手書きできる人だけが、ライブラリの挙動を推論できます**。

## 門番を通す — as を捨てて zod で検証する

上のコードには 1 箇所ずるがあります: `data as Review[]`。
[as は検査ではなく検査の免除](../../typescript-fable-101/chapters/13_advanced_types.md)でした。
新聞社の API が仕様変更で `stars` を文字列にしたら、型は通ったまま実行時に画面が壊れます。
[TS 第 14 章の門番](../../typescript-fable-101/chapters/14_runtime_validation.md)を立てましょう:

```bash
npm install zod
```

```tsx
import { z } from "zod";

const ReviewSchema = z.object({
  id: z.number(),
  critic: z.string(),
  stars: z.number().min(0).max(5),
  comment: z.string(),
});
const ReviewsSchema = z.array(ReviewSchema);
type Review = z.infer<typeof ReviewsSchema>[number];   // 型はスキーマから導出

// effect 内の該当行を差し替え:
const data: unknown = await res.json();
const parsed = ReviewsSchema.safeParse(data);
if (!parsed.success) throw new Error("劇評データの様式が不正です");
if (!ignore) setState({ status: "success", reviews: parsed.data });
```

これで「ネットワーク越しの城壁の外」から入るデータは、必ず検問を通ります。
不正データは **エラー局面として画面に正直に表示** され、深夜の白画面クラッシュには
なりません。

## ⚔️ 完成コード: `src/App.tsx`

型・スキーマ・`ReviewCorner` は上記の通り。仕上げに再試行を付けます:

```tsx
// Reactive Theater — 14 日目: 劇評コーナー(再試行付き)

function ReviewCorner() {
  const [state, setState] = useState<FetchState>({ status: "loading" });
  const [attempt, setAttempt] = useState(0);          // 再試行カウンタ

  useEffect(() => {
    let ignore = false;
    setState({ status: "loading" });                  // 再試行のたびに読み込み中へ戻す

    async function load() {
      try {
        const res = await fetch("/reviews.json");
        if (!res.ok) throw new Error(`HTTP ${res.status}`);
        const parsed = ReviewsSchema.safeParse(await res.json());
        if (!parsed.success) throw new Error("劇評データの様式が不正です");
        if (!ignore) setState({ status: "success", reviews: parsed.data });
      } catch (err) {
        if (!ignore) {
          setState({ status: "error", message: err instanceof Error ? err.message : "不明" });
        }
      }
    }
    load();
    return () => { ignore = true; };
  }, [attempt]);                                      // attempt が変わったら取得し直す

  switch (state.status) {
    case "loading":
      return <p>📰 劇評を取り寄せ中…</p>;
    case "error":
      return (
        <div>
          <p>⚠️ {state.message}</p>
          <button onClick={() => setAttempt((n) => n + 1)}>もう一度取り寄せる</button>
        </div>
      );
    case "success": {
      const avg = state.reviews.reduce((s, r) => s + r.stars, 0) / state.reviews.length;
      return (
        <section>
          <h2>📰 劇評コーナー(平均 ⭐{avg.toFixed(1)})</h2>
          <ul>
            {state.reviews.map((r) => (
              <li key={r.id}>
                {"⭐".repeat(r.stars)} {r.comment} — <em>{r.critic}</em>
              </li>
            ))}
          </ul>
        </section>
      );
    }
  }
}

function App() {
  return (
    <main>
      <h1>🎭 Reactive Theater</h1>
      <ReviewCorner />
    </main>
  );
}

export default App;
```

💡 「再試行」を `setAttempt((n) => n + 1)` という **state の変更** で表現している点に
注目してください。effect の依存配列に `attempt` を入れることで、「ボタン → state 変更 →
再レンダリング → 依存が変わったので effect 再実行」という React の正規ルートに乗せています。

## 📝 今日の舞台稽古(演習)

1. ブラウザの開発者ツール → Network タブで「Slow 4G」を選んで読み込み中表示を、`fetch("/reviewz.json")`(存在しない URL)でエラー表示と再試行を確認してください。3 局面すべてを自分の目で見ること。
2. `public/reviews.json` の 1 件の `stars` を `"5"`(文字列)に書き換え、zod の門番がエラー局面に導くことを確認してください。`as Review[]` のままだと何が起きるかも比較を。
3. `showId` を props で受け取り `fetch(\`/reviews-\${showId}.json\`)` する版に改造してください(JSON を 2 つ用意)。切り替え時に `ignore` を消すと競合状態が起こりうる理由を、シーケンスを書いて説明してください。
4. `AbortController` を調べて、クリーンアップで `controller.abort()` する版に書き換えてください(`fetch(url, { signal: controller.signal })`)。

---

次章、劇場が大所帯になってきたので **上演の無駄** を点検します。memo・useMemo・
useCallback——「再上演を省く」道具たちと、それを **使うべきでない** 場面の見極めです。
→ [第15章 無駄な再上演を減らす](15_performance.md)
