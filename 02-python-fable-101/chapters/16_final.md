# 第16章 卒業制作 — テスト、プロジェクト構成、そして出荷

## 🏪 最後のお話

Pythonic Potions は、看板 1 枚から始まって、調合・自動帳簿・セーブ・非同期接客・
プラグイン機構を備えた大繁盛店になりました。

最終章のテーマは「**壊さずに育て続けるための仕組み**」です。
どんな名店も、改装のたびに壁が崩れるようでは長続きしません。

## pytest — お店の品質検査官

**テスト** とは「コードが約束どおり動くことを、コードで確認する」ことです。
デファクトスタンダードの `pytest` を使います。

```bash
pip install pytest
```

**`tests/test_potion.py`**

```python
import pytest
from potions.models import Potion, Inventory
from potions.errors import SoldOutError


def test_sell_reduces_stock():
    potion = Potion("回復薬", 50, stock=10)
    potion.sell(3)
    assert potion.stock == 7                 # assert 文だけ。特別な作法は不要!


def test_sell_returns_price_with_tax():
    potion = Potion("回復薬", 100, stock=1)
    assert potion.sell(1) == 110


def test_sell_more_than_stock_raises():
    potion = Potion("回復薬", 50, stock=1)
    with pytest.raises(SoldOutError):        # 「例外が出ること」もテストできる
        potion.sell(2)


def test_negative_price_rejected():
    with pytest.raises(ValueError):
        Potion("怪しい薬", -10)
```

```bash
$ pytest
==== 4 passed in 0.02s ====
```

これで **リファクタリングが怖くなくなります**。どこを改装しても、
`pytest` 一発で全室を点検できるからです。

### fixture — 毎テスト共通の開店準備

```python
@pytest.fixture
def stocked_inventory():
    """テスト用の在庫一式。各テストの前に新品が用意される。"""
    inv = Inventory()
    inv.add(Potion("回復薬", 50, 10))
    inv.add(Potion("エリクサー", 500, 1))
    return inv


def test_inventory_sell(stocked_inventory):        # 引数名で fixture を受け取る
    stocked_inventory.sell("回復薬", 2)
    assert stocked_inventory.find("回復薬").stock == 8


def test_inventory_contains(stocked_inventory):    # ここには新品が届く(前のテストの影響なし)
    assert "回復薬" in stocked_inventory           # 第9章の __contains__ もテスト!
```

### parametrize — 表でまとめて検査

```python
@pytest.mark.parametrize("price, count, expected", [
    (100, 1, 110),
    (100, 3, 330),
    (50, 2, 110),
    (0, 5, 0),          # 境界値もきちんと
])
def test_checkout(price, count, expected):
    assert Potion("薬", price, stock=99).sell(count) == expected
```

> 💡 **良いテストの心得**
> - 1 テスト 1 関心事。名前は「何を保証するか」が読める文に
> - 境界値(0、1、満杯、空)と異常系(例外)を必ず含める
> - テストしにくいコードは設計が悪いサイン。第4章の「受け取って返す」関数はテストが楽だったはず

## 最終形 — プロジェクト構成

```
pythonic-potions/
├── pyproject.toml           # プロジェクトの説明書 兼 設定
├── README.md
├── .venv/                   # 仮想環境(第5章)
├── src/
│   └── potions/
│       ├── __init__.py
│       ├── models.py        # Potion, Inventory(第7〜9章)
│       ├── errors.py        # 独自例外(第6章)
│       ├── brewery.py       # 醸造パイプライン(第10章)
│       ├── ledger.py        # 帳簿デコレータ(第11章)
│       ├── persistence.py   # セーブ/ロード(第12章)
│       ├── async_shop.py    # 非同期接客(第14章)
│       ├── plugins.py       # プラグイン基盤(第15章)
│       └── main.py          # 営業ループ(第3〜5章)
└── tests/
    ├── test_potion.py
    ├── test_inventory.py
    └── test_brewery.py
```

**`pyproject.toml`** — 現代 Python プロジェクトの中心ファイルです:

```toml
[project]
name = "pythonic-potions"
version = "1.0.0"
description = "魔法薬店経営シミュレータ(Python 学習の卒業制作)"
requires-python = ">=3.10"
dependencies = ["rich"]

[project.optional-dependencies]
dev = ["pytest", "mypy", "ruff"]

[project.scripts]
potions = "potions.main:main"      # pip install 後 `potions` コマンドで開店!

[build-system]
requires = ["hatchling"]
build-backend = "hatchling.build"
```

```bash
pip install -e ".[dev]"    # 開発モードでインストール(編集が即反映される)
potions                    # どこからでも開店できる!
pytest && mypy src/        # 品質検査
ruff check src/            # リンター(コードの作法チェック)も定番
```

## 全体を振り返る — 概念の地図

16 章で学んだ概念は、バラバラの知識ではなく **積み重なる階層** です。

```mermaid
flowchart BT
    subgraph L1["土台: 値と流れ"]
        V["変数・型"] --- Col["コレクション"] --- CF["制御フロー"]
    end
    subgraph L2["構造化: 名前を付けて束ねる"]
        F["関数"] --- Mod["モジュール"] --- Ex["例外"]
    end
    subgraph L3["オブジェクト指向: データ+振る舞い"]
        Cls["クラス"] --- Inh["継承・ABC"] --- Dun["特殊メソッド"]
    end
    subgraph L4["プロトコル活用: 言語に溶け込む"]
        Gen["ジェネレータ"] --- Dec["デコレータ"] --- Ctx["コンテキストマネージャ"]
    end
    subgraph L5["大規模化: 品質と拡張性"]
        Ty["型ヒント"] --- As["非同期"] --- Meta["メタプログラミング"]
    end
    L1 --> L2 --> L3 --> L4 --> L5
    L5 --> Goal["🏆 テスト付きで出荷できる<br/>Pythonic Potions v1.0"]
```

伏線も回収しておきましょう:

| 章 | 撒かれた伏線 | 回収された章 |
|---|---|---|
| 1 | 「変数はラベル」 | 2(リストの共有)、4(引数の受け渡し) |
| 4 | 「関数も値」「Enclosing スコープ」 | 11(クロージャ → デコレータ) |
| 6 | try/finally | 12(with はその省略形) |
| 7 | `@property` という謎の記号 | 11(デコレータ)、15(ディスクリプタが正体) |
| 9 | 「演算子は dunder の化粧」 | 10(for の正体)、12(with の正体) |
| 10 | yield の凍結・再開 | 12(@contextmanager)、14(コルーチン) |
| 8 | ダックタイピング | 13(Protocol で型を与える) |

## 卒業試験(最終演習)

1. **テスト網羅**: `tests/` を作り、第9章の調合(`__add__`)・第10章のパイプライン・第15章のプラグイン登録にテストを書いてください。目標 15 テスト以上。
2. **CI ごっこ**: `pytest && mypy src/ && ruff check src/` を 1 コマンドで回すシェルスクリプト(または `Makefile`)を書いてください。
3. **自由増築**: 好きな機能を 1 つ設計から実装まで通しでやってください。おすすめ:
   - 客足シミュレーション(`random` + 非同期で自動のお客さんが来店)
   - 価格変動市場(需要に応じて価格が動く。property の門番が活躍)
   - Web API 化(`FastAPI` を調べて `GET /potions` を作る — 型ヒントがそのまま API 仕様になる感動を味わえます)

## 🎓 卒業、おめでとうございます!

看板 1 枚(`shop_name = "Pythonic Potions"`)から始まった旅は、
メタクラスがクラスを製造する深淵まで到達しました。

### この先の冒険地図

- **Web 開発**: FastAPI(型ヒント直結)、Django(第15章の魔法が満載)
- **データ分析・機械学習**: pandas、polars、scikit-learn
- **自動化・CLI**: click / typer、rich
- **さらに深く**: 『Fluent Python』(本教材の上級概念を極める定番書)、CPython のソースコードリーディング
- **写経より実戦**: 自分の困りごとを 1 つ、Python で解決してみてください。それが最高の第17章です

店主としてのあなたの Python は、もう **読める・書ける・設計できる** 段階にあります。
よい魔法薬ライフを!🧪✨

---

[← 目次に戻る](../README.md)
