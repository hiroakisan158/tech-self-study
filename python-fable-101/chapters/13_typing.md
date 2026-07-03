# 第13章 商品仕様書 — 型ヒントと静的型検査

## 🌳 上級編へようこそ

お店のコードは 500 行を超えました。ある日、手伝いに来た弟子が聞きます。

「師匠、`sell(item, count)` の `item` って、文字列ですか? `Potion` オブジェクトですか?」

…どっちだったか、あなたも一瞬迷いませんでしたか?
第1章で「変数に型はなく、値に型がある」と学びました。その自由さは、
コードが育つと「**暗黙の約束を全部暗記しておく**」負担に変わります。

**型ヒント** は、この約束を書面(仕様書)にする仕組みです。

## 基本の型ヒント

```python
def checkout(price: int, count: int = 1, tax_rate: float = 0.1) -> int:
    return int(price * count * (1 + tax_rate))

shop_name: str = "Pythonic Potions"
```

- `引数: 型` と `-> 返り値の型` を書くだけ
- **実行時には何のチェックもされません**(`checkout("a", "b")` は普通に実行されて落ちる)
- では誰が読むのか? → **エディタ、静的型検査ツール、そして人間**

```mermaid
flowchart LR
    Code["型ヒント付きコード"] --> E["エディタ 🖊️<br/>補完・その場で警告"]
    Code --> M["mypy / pyright 🔍<br/>実行せずに矛盾を検出"]
    Code --> H["人間 👀<br/>仕様書として読む"]
    M --> CI["CI で自動チェック<br/>(第16章)"]
```

### mypy — 実行せずにバグを見つける

```bash
pip install mypy
mypy shop/
```

```python
def sell(self, name: str, count: int = 1) -> int: ...

inventory.sell("回復薬", "3")   # mypy: error: Argument 2 has incompatible type "str"
```

**実行しなくても** 型の矛盾を全ファイル一斉に検査できます。
テストが通らないバグの一群を、書いた瞬間に潰せるのが静的型検査の価値です。

## コレクションと Optional

```python
def cheap_items(prices: dict[str, int], limit: int) -> list[str]:
    return [name for name, p in prices.items() if p < limit]

def find(self, name: str) -> Potion | None:
    """見つからなければ None。呼ぶ側に None 処理を強制できる!"""
    return self._potions.get(name)

potion = inventory.find("回復薬")
potion.sell()          # mypy: error! None かもしれないのに .sell した
if potion is not None:
    potion.sell()      # OK: この分岐の中では Potion だと mypy も理解する(型の絞り込み)
```

| 書き方(3.10+) | 意味 |
|---|---|
| `list[Potion]` | Potion のリスト |
| `dict[str, int]` | str → int の辞書 |
| `tuple[str, ...]` | str が何個でも並ぶタプル |
| `int \| None` | int または None(旧 `Optional[int]`) |
| `int \| str` | int または str(旧 `Union[int, str]`) |
| `Literal["gold", "card"]` | この 2 つの文字列だけ |
| `Callable[[int], str]` | int を受け取り str を返す関数 |

`Literal` は支払い方法のような「選択肢が決まっている文字列」に最適です:

```python
from typing import Literal

PayMethod = Literal["gold", "card", "barter"]   # 型に別名を付けられる

def pay(amount: int, method: PayMethod = "gold") -> None: ...

pay(100, "cash")   # mypy: error! "cash" は選択肢にない(タイポも一網打尽)
```

## TypedDict — dict に仕様書を付ける

第12章のセーブデータのような「形の決まった dict」には `TypedDict`:

```python
from typing import TypedDict

class PotionRow(TypedDict):
    name: str
    price: int
    stock: int

def load_row(row: PotionRow) -> Potion:
    return Potion(row["name"], row["price"], row["stock"])
    # row["prise"] とタイポすれば mypy が即座に指摘!
```

## Protocol — ダックタイピングに型を与える

第8章で「`use()` さえあれば動く」ダックタイピングを学びました。
**Protocol** はその「〜さえあれば」を型として表現します。**継承は不要** です。

```python
from typing import Protocol

class Sellable(Protocol):
    """「売り物」の条件: name と price があり、sell できること。"""
    name: str
    price: int
    def sell(self, count: int = 1) -> int: ...


def bulk_sell(items: list[Sellable], count: int = 1) -> int:
    return sum(item.sell(count) for item in items)
```

`Potion` は `Sellable` を継承していませんが、条件を満たしているので
`bulk_sell` に渡せます(**構造的部分型**)。将来「魔法の杖」クラスを追加しても、
`name` / `price` / `sell` さえ持てば無改造で受け入れられます。

```mermaid
flowchart TD
    P["Sellable(Protocol)<br/>📜 name / price / sell を持つこと"]
    A["Potion 🧪"] -.->|条件を満たす<br/>継承は不要| P
    B["MagicWand 🪄"] -.->|条件を満たす| P
    C["Ticket 🎫<br/>(sell がない)"] -.->|"❌ mypy が拒否"| P
    F["bulk_sell(items: list[Sellable])"] --> P
```

- **ABC(第8章)**: 「この一族であること」を要求(名目的)。実行時にも強制できる
- **Protocol**: 「この形であること」を要求(構造的)。ライブラリ間の疎結合に強い

## ジェネリクス — 型を後から差し込む

「ポーション専用の棚」「材料専用の棚」…中身の型ごとに棚クラスを書くのは無駄です。
**型引数** を持つクラス(ジェネリクス)にしましょう(3.12+ の新文法):

```python
class Shelf[T]:
    """何でも載る棚。ただし 1 つの棚には 1 種類だけ。"""
    def __init__(self) -> None:
        self._items: list[T] = []

    def put(self, item: T) -> None:
        self._items.append(item)

    def take(self) -> T:
        return self._items.pop()


potion_shelf: Shelf[Potion] = Shelf()
potion_shelf.put(healing)          # OK
potion_shelf.put("ただの文字列")    # mypy: error!

taken = potion_shelf.take()        # taken は Potion 型だと mypy が知っている
taken.use()                        # 補完も効く!
```

関数にも使えます:

```python
def first_in_stock[T: Potion](potions: list[T]) -> T | None:
    """在庫のある最初の 1 本。渡した型がそのまま返り値の型になる。"""
    for p in potions:
        if p.stock > 0:
            return p
    return None

# list[HealingPotion] を渡せば返り値も HealingPotion | None になる
```

`[T: Potion]` は「T は Potion かその子孫」という **上限付き型引数** です。
(3.11 以前は `from typing import TypeVar; T = TypeVar("T", bound=Potion)` と書きます)

## 型ヒントとの付き合い方

- **公開 API(モジュール間で呼び合う関数・メソッド)には必ず書く**。仕様書だから
- 3 行のローカル変数にまで書かなくてよい。mypy は推論が得意
- 既存コードには少しずつ。`mypy --strict` は最終目標であってスタート地点ではない
- 型が複雑になりすぎたら **設計が複雑すぎるサイン** かもしれません

## 🧪 完成コード: 型付き `models.py`(抜粋)

```python
from typing import Protocol
from errors import SoldOutError


class Sellable(Protocol):
    name: str
    price: int
    def sell(self, count: int = 1) -> int: ...


class Inventory:
    def __init__(self) -> None:
        self._potions: dict[str, Potion] = {}

    def add(self, potion: Potion) -> None:
        self._potions[potion.name] = potion

    def find(self, name: str) -> Potion | None:
        return self._potions.get(name)

    def sell(self, name: str, count: int = 1) -> int:
        potion = self.find(name)
        if potion is None:
            raise KeyError(name)
        return potion.sell(count)

    def __iter__(self):
        yield from self._potions.values()
```

## 📝 今日の開店準備(演習)

1. これまでの `shop/` 全ファイルに型ヒントを付け、`mypy shop/` がエラーゼロになるまで直してください(1 つはタイポが見つかるはず…?)。
2. `Usable` Protocol(`use() -> str` を持つ)を定義し、`try` コマンドの処理関数の引数型にしてください。
3. `Shelf[T]` に `__iter__` と `__len__` を型付きで実装してください(第9章の復習)。

---

仕様書は完璧。しかしお店の前には行列が…。配達妖精の帰りを **待っている間**、
レジは完全に止まっています。「待ち時間に別の仕事をする」技術へ
→ [第14章 行列をさばく](14_async.md)
