# Design Document

## Overview

cronja は cron 式と日本語テキストを相互変換するツールです。コアは副作用のない純粋関数群（標準ライブラリのみ）として実装し、同一コアを CLI と MCP サーバーの両方から呼び出します。

### 主要機能

| 関数 | 入力 → 出力 |
|---|---|
| `parse(s)` | cron 文字列 → `CronExpr \| ParseError` |
| `format(x)` | `CronExpr` → cron 文字列（正規形） |
| `describe(x)` | `CronExpr` → 日本語説明文 |
| `from_text(s)` | 日本語テキスト → `CronExpr \| ParseError` |
| `next_runs(x, base, n)` | `CronExpr`, `datetime`, `int` → `list[datetime] \| SchedulerError` |
| `lint(x, base)` | `CronExpr`, `datetime` → `list[LintWarning]` |

### 設計方針

- **エラーは戻り値で返す**: 例外を外部に伝播させない。失敗しうる関数は結果の型とエラーの型の union を返す。
- **不変データ**: `CronExpr` は `@dataclass(frozen=True)`。フィールド値はすべて `frozenset[int]`。
- **型ヒント必須**: すべての関数の引数・戻り値、すべての dataclass フィールドに型ヒントを付ける。
- **naive datetime のみ**: タイムゾーン処理なし。コアは時計を読まず、現在時刻は引数 `base` で受け取る。
- **UTF-8 固定**: CLI・MCP サーバーの起動時に stdin/stdout/stderr を UTF-8 に設定する。
- **describe と from_text は同じ構造**: どちらも「年の部分 → 日付の部分 → 時刻の部分」の 3 つに分けて処理する。片方だけが知っている言い回しを作らない。

---

## Architecture

```
┌──────────────────────────────────────────────────┐
│                インターフェース層                    │
│   cli.py（argparse）       mcp_server.py（mcp）     │
└───────────────┬──────────────────┬───────────────┘
                │                  │
┌───────────────▼──────────────────▼───────────────┐
│                コア層（副作用なし）                  │
│  parser.py  formatter.py  describe.py             │
│  from_text.py  schedule.py  lint.py               │
│                    model.py                       │
└──────────────────────────────────────────────────┘
```

コア層は標準ライブラリのみを使用します。`mcp` ライブラリに依存するのは `mcp_server.py` だけです。コア層のモジュールは `cli`、`mcp_server`、`argparse`、`mcp` を import せず、print もしません。

---

## Components and Interfaces

### モジュール構成（src レイアウト）

モジュール名は `.kiro/steering/structure.md` に合わせます。

```
src/
  cronja/
    __init__.py       # コアの公開 API を再エクスポート
    model.py          # CronExpr, ParseError, SchedulerError, LintWarning, フィールド定数
    parser.py         # parse(s) -> CronExpr | ParseError
    formatter.py      # format(x) -> str
    describe.py       # describe(x) -> str
    from_text.py      # from_text(s) -> CronExpr | ParseError
    schedule.py       # next_runs(x, base, n), matches(x, dt)
    lint.py           # lint(x, base) -> list[LintWarning]
    cli.py            # main() エントリポイント `cronja`
    mcp_server.py     # main() エントリポイント `cronja-mcp`
tests/
  strategies.py                  # 共通の Hypothesis ストラテジーと検証用の補助関数
  test_roundtrip_properties.py   # プロパティ 1〜4
  test_schedule_properties.py    # プロパティ 5〜7
  test_robustness_properties.py  # プロパティ 8
  test_parser.py
  test_formatter.py
  test_describe.py
  test_from_text.py
  test_schedule.py
  test_lint.py
  test_cli.py
pyproject.toml
README.md
```

### 各モジュールの公開インターフェース

#### `model.py`

```python
from dataclasses import dataclass

FIELD_RANGES: dict[str, range] = {
    "second": range(0, 60),
    "minute": range(0, 60),
    "hour":   range(0, 24),
    "day":    range(1, 32),
    "month":  range(1, 13),
    "dow":    range(0, 7),      # 0=日曜
    "year":   range(1970, 2100),
}

FULL_SECOND: frozenset[int] = frozenset(FIELD_RANGES["second"])
FULL_MINUTE: frozenset[int] = frozenset(FIELD_RANGES["minute"])
FULL_HOUR:   frozenset[int] = frozenset(FIELD_RANGES["hour"])
FULL_DAY:    frozenset[int] = frozenset(FIELD_RANGES["day"])
FULL_MONTH:  frozenset[int] = frozenset(FIELD_RANGES["month"])
FULL_DOW:    frozenset[int] = frozenset(FIELD_RANGES["dow"])
FULL_YEAR:   frozenset[int] = frozenset(FIELD_RANGES["year"])
ZERO_SECOND: frozenset[int] = frozenset({0})

@dataclass(frozen=True)
class CronExpr:
    second: frozenset[int]
    minute: frozenset[int]
    hour:   frozenset[int]
    day:    frozenset[int]
    month:  frozenset[int]
    dow:    frozenset[int]
    year:   frozenset[int]

@dataclass(frozen=True)
class ParseError:
    field: str | None   # 問題のあるフィールド名。特定できない場合は None
    value: str          # 問題のある文字列
    reason: str         # 日本語の説明

@dataclass(frozen=True)
class SchedulerError:
    reason: str         # 日本語の説明

@dataclass(frozen=True)
class LintWarning:
    code: str           # 警告種別の識別子（例: "OR_CONDITION"）
    message: str        # 日本語の説明
```

- 等価性はフィールドの集合の等価性で決まる。フラグ（`day_is_star` など）は持たない。
- 「全範囲」かどうかは `x.day == FULL_DAY` のように集合の比較で判定する。
- 警告の型名は、組み込みの `Warning` と重ならないよう `LintWarning` とする。
- 「ステップ集合」（要件の用語集）の判定は `model.py` に 1 つだけ置き、formatter・describe・lint の 3 か所から同じ関数を使う。

```python
def step_of(values: frozenset[int], field: str) -> int | None:
    """values がステップ集合ならステップ値 n を返す。そうでなければ None。"""
    rng = FIELD_RANGES[field]
    for n in range(2, len(rng)):
        if values == frozenset(range(rng.start, rng.stop, n)) and (
            len(values) >= 3 or len(rng) % n == 0
        ):
            return n
    return None
```

#### `parser.py`

```python
def parse(s: str) -> CronExpr | ParseError: ...
```

#### `formatter.py`

```python
def format(x: CronExpr) -> str: ...
```

公開名は要件に合わせて `format` とする。組み込みの `format` と同名なので、ほかのモジュールからは `from cronja import formatter` として `formatter.format(x)` の形で呼ぶ。

#### `describe.py`

```python
def describe(x: CronExpr) -> str: ...
```

#### `from_text.py`

```python
def from_text(s: str) -> CronExpr | ParseError: ...
```

#### `schedule.py`

```python
def next_runs(x: CronExpr, base: datetime, n: int) -> list[datetime] | SchedulerError: ...
def matches(x: CronExpr, dt: datetime) -> bool: ...
```

#### `lint.py`

```python
def lint(x: CronExpr, base: datetime) -> list[LintWarning]: ...
```

---

## Data Models

### describe() の出力 BNF

`describe()` が出力する日本語テキストの文法です。`from_text()` は、この文法に従う文をすべて `CronExpr` に戻せます。

```bnf
<description>  ::= <body>
                 | <body> <year_suffix>

<body>         ::= <time_repeat>                (* 日付が「毎日」で、時刻が繰り返しの言い回し *)
                 | "毎日" <time_exact>           (* 「の」を入れない *)
                 | <date> "の" <time>            (* それ以外 *)

(* ---------- 日付の部分 ---------- *)
<date>         ::= "毎日"
                 | "平日"
                 | "土日"
                 | "毎週" <dow_enum> "曜"
                 | "毎月" <int_enum> "日"
                 | "毎月" <int_enum> "日または" <dow_enum> "曜"
                 | <int_enum> "月の" <month_body>

<month_body>   ::= "毎日"
                 | <int_enum> "日"
                 | <dow_enum> "曜"
                 | <int_enum> "日または" <dow_enum> "曜"

(* ---------- 時刻の部分 ---------- *)
<time>         ::= <time_repeat> | <time_exact> | <time_regular>

<time_repeat>  ::= "毎秒"
                 | <int> "秒おき"
                 | "毎分"
                 | <int> "分おき"
                 | "毎時" <int> "分"

<time_exact>   ::= <int> "時" <int> "分"

<time_regular> ::= <hour_part> <minute_part> <second_part>
<hour_part>    ::= "毎時" | <int_enum> "時"
<minute_part>  ::= "毎分" | <int_enum> "分"
<second_part>  ::= "" | "毎秒" | <int_enum> "秒"

(* ---------- 年の部分 ---------- *)
<year_suffix>  ::= "（対象年: " <int_enum> "）"

(* ---------- 列挙 ---------- *)
<int_enum>     ::= <int_item> ("・" <int_item>)*
<int_item>     ::= <int> | <int> "〜" <int>
<dow_enum>     ::= <dow_item> ("・" <dow_item>)*
<dow_item>     ::= <dow_name> | <dow_name> "〜" <dow_name>
<dow_name>     ::= "日" | "月" | "火" | "水" | "木" | "金" | "土"
<int>          ::= [0-9]+
```

**列挙の規則**

- 値は昇順に並べる。曜日の順は 日(0)・月(1)・火(2)・水(3)・木(4)・金(5)・土(6)。
- 3 つ以上連続する値は `a〜b` とまとめる。2 つ以下の連続はまとめずに「・」で並べる。
  - 例: `{9,10,11,12}` → `9〜12`、`{1,15}` → `1・15`、`{1,2}` → `1・2`、`{1,2,3,10}` → `1〜3・10`
  - 例: 曜日 `{1,3,5}` → `月・水・金`、曜日 `{1,2,3,4}` → `月〜木`
- この規則により、1 つの集合に対する表記は 1 通りに決まる。

**`<time_regular>` の意味**

- `<hour_part>` の「毎時」は時が全範囲、`<minute_part>` の「毎分」は分が全範囲、`<second_part>` の「毎秒」は秒が全範囲であることを表す。
- `<second_part>` が空の場合、秒は `{0}`。
- 時・分・秒の組み合わせは、cron と同じく直積（すべての組み合わせ）を表す。
- 「毎時15分」と「9時0分」は `<time_regular>` としても同じ意味に読める。表記と意味が一致しているので、どちらの規則で読んでも結果は変わらない。

### describe() の生成規則

`describe(x)` は、年の部分・日付の部分・時刻の部分を独立に作り、最後に組み合わせます。

**日付の部分**（要件 3.3、3.4。上から順に判定し、最初に該当した行を使う）

| 順 | 条件 | 出力 |
|---|---|---|
| 1 | 月・日・曜日がすべて全範囲 | `毎日` |
| 2 | 月・日が全範囲、曜日が `{1,2,3,4,5}` | `平日` |
| 3 | 月・日が全範囲、曜日が `{0,6}` | `土日` |
| 4 | 月・日が全範囲、曜日が制限されている | `毎週<dow_enum>曜` |
| 5 | 月・曜日が全範囲、日が制限されている | `毎月<int_enum>日` |
| 6a | 月が全範囲、日と曜日の両方が制限されている | `毎月<int_enum>日または<dow_enum>曜` |
| 6b | 月が制限されている、日・曜日が全範囲 | `<int_enum>月の毎日` |
| 6c | 月が制限されている、日だけが制限されている | `<int_enum>月の<int_enum>日` |
| 6d | 月が制限されている、曜日だけが制限されている | `<int_enum>月の<dow_enum>曜` |
| 6e | 月が制限されている、日と曜日の両方が制限されている | `<int_enum>月の<int_enum>日または<dow_enum>曜` |

**時刻の部分**（要件 3.5。上から順に判定し、最初に該当した行を使う）

| 順 | 条件 | 出力 | 種別 |
|---|---|---|---|
| 1 | 秒・分・時がすべて全範囲 | `毎秒` | 繰り返し |
| 2 | 分・時が全範囲、秒がステップ集合（ステップ値 n） | `n秒おき` | 繰り返し |
| 3 | 秒が `{0}`、分・時が全範囲 | `毎分` | 繰り返し |
| 4 | 秒が `{0}`、時が全範囲、分がステップ集合（ステップ値 n） | `n分おき` | 繰り返し |
| 5 | 秒が `{0}`、時が全範囲、分が単一値 `{m}` | `毎時m分` | 繰り返し |
| 6 | 秒が `{0}`、時が単一値 `{h}`、分が単一値 `{m}` | `h時m分` | 特定時刻 |
| 7 | 上のどれにも該当しない | `<time_regular>` | 規則的 |

**年の部分**（要件 3.6）

- 年が全範囲: 空文字列
- 年が制限されている: `（対象年: <int_enum>）`

**組み合わせ**（要件 3.7）

```python
def describe(x: CronExpr) -> str:
    date_part, time_part, time_kind = _describe_date(x), *_describe_time(x)
    if date_part == "毎日" and time_kind == "repeat":
        body = time_part                    # 日付の部分を省略
    elif date_part == "毎日" and time_kind == "exact":
        body = date_part + time_part        # 「の」を入れない
    else:
        body = date_part + "の" + time_part
    return body + _describe_year(x)
```

**出力例**

| cron 式 | 説明文 |
|---|---|
| `* * * * *` | 毎分 |
| `15 * * * *` | 毎時15分 |
| `*/30 * * * *` | 30分おき |
| `0 9 * * *` | 毎日9時0分 |
| `0 9 * * 1` | 毎週月曜の9時0分 |
| `0 9 * * 1-5` | 平日の9時0分 |
| `*/30 * * * 1-5` | 平日の30分おき |
| `0 9 1 * *` | 毎月1日の9時0分 |
| `*/10 * * * * *` | 10秒おき |
| `0 0 9 * * 1 2027` | 毎週月曜の9時0分（対象年: 2027） |
| `0 9-17 * * *` | 毎日の9〜17時0分 |
| `5,35 * * * *` | 毎日の毎時5・35分 |
| `0 9 * * 1,3,5` | 毎週月・水・金曜の9時0分 |
| `0 9 1,15 * *` | 毎月1・15日の9時0分 |
| `0 9 13 * 5` | 毎月13日または金曜の9時0分 |
| `0 0 * 3 *` | 3月の毎日の0時0分 |
| `30 0 9 * * *` | 毎日の9時0分30秒 |
| `0 */30 * * * * 2027-2030` | 30分おき（対象年: 2027〜2030） |
| `0,45 * * * *` | 毎日の毎時0・45分 |
| `0 0 30 2 *` | 2月の30日の0時0分 |

---

## 主要アルゴリズム

### parser.py: フィールド展開

1. `s.split()` でフィールドに分割し、フィールド数（5・6・7）を確認する。それ以外は `ParseError`。
2. フィールド数に応じて、秒を `{0}`、年を全範囲で補完する。
3. 各フィールドは `[0-9*/,-]` 以外の文字を含んでいれば `ParseError`（英字名、`L` `W` `#` `?`、全角数字を含む）。数値の変換には、ASCII の数字だけを受け付ける方法を使う。
4. `,` で分割し、各要素を次の順で解釈して和集合を取る。
   - `*/n` → `range(min, max + 1, n)`
   - `a-b/n` → `range(a, b + 1, n)`
   - `a-b` → `range(a, b + 1)`
   - `*` → 全範囲
   - 整数 → `{v}`
5. 次の検査は例外に頼らず、明示的に行う（要件 1.11、1.13、1.14）。
   - 値が有効範囲外（曜日は 0〜7 を許す） → フィールド名と値を含む `ParseError`
   - `a > b` → フィールド名と `a`・`b` を含む `ParseError`
   - ステップ `n` が 1 未満、または `n > 最大値 − 最小値` → フィールド名とステップ値を含む `ParseError`
6. 曜日の 7 を 0 に正規化する。
7. 最外層の `try/except Exception` は、想定外の入力に対する最後の受け皿としてだけ使う。

### formatter.py: フィールドの文字列化

フィールドごとに、次の優先順位で文字列にします（要件 2.5〜2.10）。

```python
def _format_field(values: frozenset[int], field: str) -> str:
    rng = FIELD_RANGES[field]
    if values == frozenset(rng):
        return "*"
    vals = sorted(values)
    if len(vals) == 1:
        return str(vals[0])
    if vals == list(range(vals[0], vals[-1] + 1)):
        return f"{vals[0]}-{vals[-1]}"
    if (n := step_of(values, field)) is not None:
        return f"*/{n}"
    if len(vals) >= 3:
        diffs = {b - a for a, b in zip(vals, vals[1:])}
        if len(diffs) == 1 and (n := diffs.pop()) >= 2:
            return f"{vals[0]}-{vals[-1]}/{n}"
    return ",".join(str(v) for v in vals)
```

フィールド数（要件 2.1〜2.3）:

- 年が全範囲、秒が `{0}` → 5 フィールド（分 時 日 月 曜日）
- 年が全範囲、秒が `{0}` 以外 → 6 フィールド（秒 分 時 日 月 曜日）
- 年が全範囲でない → 7 フィールド（秒 分 時 日 月 曜日 年）

### from_text.py: 3 段階の決定的パーサ

`describe()` と同じ構造で、年 → 日付 → 時刻の順に読みます。LLM は使いません。

**前処理**

1. `unicodedata.normalize("NFKC", s)` で正規化する。全角数字は半角に、`（` `）` `：` は `(` `)` `:` に、`～`（U+FF5E）は `~` になる。
2. 空白文字をすべて取り除く。
3. 空文字列になったら `ParseError`。

以降の規則は、正規化後の文字列に対して適用します。範囲の区切りは `〜`（U+301C）と `~` の両方を受け付け、列挙の区切りは `・` `,` `、` を受け付けます。

**手順**

1. **年**: 末尾が `(対象年:<int_enum>)` に一致すれば、その部分を切り離して年の集合にする。なければ年は全範囲。
2. **本体**: 残りの文字列を次の順に試す。
   1. 全体が時刻の部分として読める → 日付は「毎日」（要件 4.8）
   2. `毎日` で始まり、残りが時刻の部分として読める → 日付は「毎日」
   3. 最後の `の` で 2 つに分け、左を日付の部分、右を時刻の部分として読む（時刻の部分は `の` を含まないので、最後の `の` で分ければよい）
   4. どれにも当てはまらなければ `ParseError`
3. 列挙を展開した値が有効範囲外なら `ParseError`。
4. 最外層の `try/except Exception` は最後の受け皿としてだけ使う（要件 4.5）。

**時刻の部分の読み方**（`_parse_time`。上から順に試す）

| 形 | 結果（秒 / 分 / 時） |
|---|---|
| `毎秒` | 全範囲 / 全範囲 / 全範囲 |
| `n秒おき`（1≤n≤59） | `range(0, 60, n)` / 全範囲 / 全範囲 |
| `毎分` | `{0}` / 全範囲 / 全範囲 |
| `n分おき`（1≤n≤59） | `{0}` / `range(0, 60, n)` / 全範囲（「45分おき」は分 `{0, 45}` になる。ステップ集合ではないので、`describe` は「毎時0・45分」と出力する） |
| `<hour_part><minute_part><second_part>` | BNF の `<time_regular>` のとおり |
| `h時`（分なし） | `{0}` / `{0}` / `{h}`（要件 4.7） |
| `h:mm` | `{0}` / `{mm}` / `{h}` |
| `午前h時[m分]` | 時は `h`（午前12時は 0 時） |
| `午後h時[m分]` | 時は `h + 12`（午後12時は 12 時） |

「毎時15分」と「9時0分」は `<time_regular>` の行で読めます。

**日付の部分の読み方**（`_parse_date`。上から順に試す）

| 形 | 結果（日 / 月 / 曜日） |
|---|---|
| `毎日` | 全範囲 / 全範囲 / 全範囲 |
| `平日` | 全範囲 / 全範囲 / `{1,2,3,4,5}` |
| `土日`、`週末` | 全範囲 / 全範囲 / `{0,6}` |
| `毎月<int_enum>日または<dow_enum>曜` | 日の集合 / 全範囲 / 曜日の集合 |
| `毎月<int_enum>日` | 日の集合 / 全範囲 / 全範囲 |
| `<int_enum>月の<month_body>` | BNF の `<month_body>` のとおり / 月の集合 |
| `[毎週]<dow_enum>[曜\|曜日]` | 全範囲 / 全範囲 / 曜日の集合（「月〜金」「毎週月曜日」も読める。要件 4.3） |

### schedule.py: スキップアヘッド

1 秒ずつ進める総当たりはせず、一致しない単位を丸ごと飛ばします。**飛んだあとは必ずループの先頭に戻り、年から判定し直します。**

```python
LIMIT_YEAR = 2099

def next_runs(x: CronExpr, base: datetime, n: int) -> list[datetime] | SchedulerError:
    if n < 1:
        return SchedulerError("件数は 1 以上で指定してください")
    t = base.replace(microsecond=0) + timedelta(seconds=1)   # マイクロ秒を切り捨てる
    if t.year < 1970:
        t = datetime(1970, 1, 1)
    results: list[datetime] = []
    while len(results) < n and t.year <= LIMIT_YEAR:
        if t.year not in x.year:
            later = [y for y in x.year if y > t.year]
            if not later:
                break
            t = datetime(min(later), 1, 1)
            continue
        if t.month not in x.month:
            later = [m for m in x.month if m > t.month]
            t = datetime(t.year, min(later), 1) if later else datetime(t.year + 1, 1, 1)
            continue
        if not _day_matches(x, t):
            t = datetime(t.year, t.month, t.day) + timedelta(days=1)
            continue
        if t.hour not in x.hour:
            later = [h for h in x.hour if h > t.hour]
            t = (t.replace(hour=min(later), minute=0, second=0) if later
                 else datetime(t.year, t.month, t.day) + timedelta(days=1))
            continue
        if t.minute not in x.minute:
            later = [m for m in x.minute if m > t.minute]
            t = (t.replace(minute=min(later), second=0) if later
                 else t.replace(minute=0, second=0) + timedelta(hours=1))
            continue
        if t.second not in x.second:
            later = [s for s in x.second if s > t.second]
            t = (t.replace(second=min(later)) if later
                 else t.replace(second=0) + timedelta(minutes=1))
            continue
        results.append(t)
        t += timedelta(seconds=1)
    if not results:
        return SchedulerError("2099年末までに実行時刻が見つかりませんでした")
    return results
```

日と曜日の判定（要件 5.5、5.6）:

```python
def _day_matches(x: CronExpr, dt: datetime) -> bool:
    dow = (dt.weekday() + 1) % 7          # Python は月曜=0、cron は日曜=0
    day_restricted = x.day != FULL_DAY
    dow_restricted = x.dow != FULL_DOW
    if day_restricted and dow_restricted:
        return dt.day in x.day or dow in x.dow
    if day_restricted:
        return dt.day in x.day
    if dow_restricted:
        return dow in x.dow
    return True

def matches(x: CronExpr, dt: datetime) -> bool:
    return (dt.microsecond == 0 and dt.year in x.year and dt.month in x.month
            and _day_matches(x, dt) and dt.hour in x.hour
            and dt.minute in x.minute and dt.second in x.second)
```

### lint.py: 警告条件

すべての条件を集合の比較で判定します。該当する警告をすべて返し、なければ空のリストを返します。

| コード | 条件 | メッセージ |
|---|---|---|
| `OR_CONDITION` | 日と曜日の両方が制限されている | 日と曜日の両方を指定すると、どちらかに一致した日に実行されます |
| `UNEVEN_STEP` | 秒・分・時・日・月・曜日のいずれかがステップ集合で、値の個数 % ステップ値 ≠ 0 | `{フィールド名}` の間隔が不均等になります |
| `NO_VALID_DATE` | 曜日が全範囲で、有効な月日の組み合わせが 1 つもない。または、有効な組み合わせが 2 月 29 日だけで、年にうるう年が含まれない | 指定した月日は存在しないため、実行されません |
| `LEAP_YEAR_ONLY` | 曜日が全範囲で、有効な月日の組み合わせが 2 月 29 日だけで、年にうるう年が含まれる | うるう年にしか実行されません |
| `ALL_YEARS_PAST` | 年が制限されていて、年の最大値が `base.year` 未満 | 指定した年はすべて過去のため、今後実行されません |
| `EVERY_SECOND` | 7 フィールドすべてが全範囲 | 毎秒実行になっています |
| `EVERY_MINUTE` | 秒が `{0}`、分・時・日・月・曜日が全範囲（年は問わない） | 毎分実行になっています |

- 値の個数: 秒 60、分 60、時 24、日 31、月 12、曜日 7。年は `UNEVEN_STEP` の対象外。
- 有効な月日の組み合わせ: 月 m と日 d の組のうち、d が m 月の日数以下のもの（2 月は 29 日まで）。
- うるう年: `(year % 4 == 0 and year % 100 != 0) or year % 400 == 0`

---

## インターフェース仕様

### CLI

基準時刻を決めるのは `cli.py` だけです。`--from` があればその値、なければ `datetime.now()` を秒単位に切り捨てた値を使い、コアには引数で渡します。

#### `cronja explain "<cron式>" [--from <ISO日時>]`

```
説明: 平日の9時0分
次の実行:
  2026-10-05T09:00:00
  2026-10-06T09:00:00
  2026-10-07T09:00:00
警告: なし
```

警告がある場合は、`警告:` の下に 1 行ずつメッセージを表示します。実行時刻が見つからない場合:

```
説明: 2月の30日の0時0分
次の実行: 実行予定はありません
警告:
  指定した月日は存在しないため、実行されません
```

#### `cronja make "<日本語>"`

```
*/30 * * * 1-5
```

#### `cronja next "<cron式>" [--count N] [--from <ISO日時>]`

`--count` の既定値は 1、有効範囲は 1〜100。1 行に 1 件ずつ表示します。

```
2026-10-05T09:00:00
```

### MCP ツール

#### `explain_cron`

入力: `cron_expression`（必須、文字列）、`from_time`（省略可、`YYYY-MM-DDTHH:MM:SS`。省略時はサーバーの現在時刻）

```json
{
  "description": "毎月13日または金曜の9時0分",
  "next_runs": ["2026-10-09T09:00:00", "2026-10-13T09:00:00", "2026-10-16T09:00:00"],
  "warnings": [
    {"code": "OR_CONDITION", "message": "日と曜日の両方を指定すると、どちらかに一致した日に実行されます"}
  ]
}
```

実行時刻が見つからない場合は `isError` を立てず、`next_runs` を空のリストにします。

#### `make_cron`

入力: `text`（必須、文字列）

```json
{"cron_expression": "*/30 * * * 1-5"}
```

#### `next_runs`

入力: `cron_expression`（必須）、`count`（省略可、整数、既定 1、1〜100）、`from_time`（省略可）

```json
{"runs": ["2026-10-05T09:00:00"]}
```

#### エラー時の出力

`isError: true` とし、内容のテキストに次の JSON を入れます。`field` と `value` は該当しない場合 `null` です。

```json
{"error": {"field": "minute", "value": "99", "reason": "分の値が範囲外です: 99（0〜59）"}}
```

`isError: true` にする具体的な方法は、使用する `mcp` SDK のバージョンに合わせて実装時に選びます。

### パッケージ構成（要件 9、11）

- `pyproject.toml`: ビルドバックエンドは `hatchling`、`requires-python = ">=3.12"`。
- `[project.scripts]`: `cronja = "cronja.cli:main"`、`cronja-mcp = "cronja.mcp_server:main"`
- `[project.dependencies]`: `mcp` のみ。`[dependency-groups]` の `dev`: `pytest`、`hypothesis`、`ruff`
- `README.md`（英語）: 概要、クイックスタート、「Kiro features used」の表（7 行）。

---

## Correctness Properties

*プロパティとは、システムの正しい動作において常に成り立つべき特性・振る舞いのことです。具体的には、あらゆる有効な実行において真であるべき命題です。プロパティは人間が読める仕様と、機械が検証できる正確性保証の橋渡しをします。*

### Property 1: cron 文字列のラウンドトリップ

*任意の*有効な `CronExpr` `x` に対して、`parse(format(x)) == x` が成り立つ。

**Validates: Requirements 2.11, 10.2**

### Property 2: 日本語のラウンドトリップ

*任意の*有効な `CronExpr` `x` に対して、`from_text(describe(x)) == x` が成り立つ。

**Validates: Requirements 3.1, 3.9, 4.1, 10.3**

### Property 3: 正規形の冪等性

*任意の*有効な `CronExpr` `x` に対して、`format(parse(format(x))) == format(x)` が成り立つ。

**Validates: Requirements 2.12, 10.4**

### Property 4: フィールド数の規則

*任意の*有効な `CronExpr` `x` に対して、`format(x)` のフィールド数が要件 2 の規則（秒が `{0}` かつ年が全範囲 → 5 フィールド、秒が `{0}` 以外かつ年が全範囲 → 6 フィールド、年が全範囲でない → 7 フィールド）に従う。

**Validates: Requirements 2.1, 2.2, 2.3, 2.4, 10.5**

### Property 5: 実行時刻の時系列正確性

*任意の*有効な `CronExpr` `x`、基準時刻 `base`（1970-01-01〜2099-12-31 の naive datetime）、件数 `n`（1〜20 の整数）に対して、`next_runs(x, base, n)` がリストを返した場合、各時刻は `base` より後で、厳密に昇順で、重複がなく、件数は `n` 以下である。

**Validates: Requirements 5.1, 5.2, 5.3, 10.6**

### Property 6: マッチング正確性

*任意の*有効な `CronExpr` `x`、基準時刻 `base`、件数 `n`（1〜20）に対して、`next_runs(x, base, n)` がリストを返した場合、各時刻は `x` の全フィールド条件（秒・分・時・月・年が集合に含まれ、日と曜日は要件 5.5 と 5.6 の規則）を満たす。

**Validates: Requirements 5.4, 5.5, 5.6, 10.7**

### Property 7: 実行時刻の取りこぼしなし

*任意の*有効な `CronExpr` `x`（時・日・月・曜日・年がすべて全範囲、秒と分は任意）と基準時刻 `base` に対して、`base` と最初の結果の間、および `next_runs(x, base, n)` の連続する 2 つの結果の間に、`x` にマッチする時刻が存在しない。この条件下では実行間隔が 1 時間以内に収まるため、間の時刻を 1 秒ずつ調べて検証できる。

**Validates: Requirements 5.7, 10.8**

### Property 8: 例外なしの堅牢性

*任意の*文字列 `s` に対して、`parse(s)` と `from_text(s)` のどちらも例外を外部に伝播させず、`CronExpr` または `ParseError` のいずれかを返す。

**Validates: Requirements 1.15, 4.5, 10.9**

---

## Error Handling

例外を外部に伝播させません。コアの各関数は union 型を返し、呼び出し側は `isinstance` または `match` で分岐します。

```python
match parse(user_input):
    case CronExpr() as expr:
        output = describe(expr)
    case ParseError() as err:
        print(f"エラー: {err.reason}", file=sys.stderr)
        sys.exit(1)
```

エラーメッセージ（`reason`）は日本語で書き、何がどこで間違っているかが分かる内容にします（例: 「フィールド数が不正です: 4（5・6・7 のいずれかで指定してください）」）。

### CLI のエラー処理

| 状況 | 動作 |
|---|---|
| `parse()` または `from_text()` が `ParseError` を返す | stderr にメッセージ、終了コード 1 |
| `explain` で `next_runs()` が `SchedulerError` を返す | 説明文と警告を表示、実行時刻の欄に「実行予定はありません」、終了コード 0 |
| `next` で `next_runs()` が `SchedulerError` を返す | stderr にメッセージ、終了コード 1 |
| `--from` が `YYYY-MM-DDTHH:MM:SS` 形式でない | stderr にメッセージ、終了コード 1 |
| `--count` が 1〜100 の範囲外、または整数でない | stderr に有効範囲を含むメッセージ、終了コード 1 |

`--count` と `--from` の検査は `argparse` の既定のエラー処理（終了コード 2）に任せず、`cli.py` で行って終了コード 1 にします。

### MCP サーバーのエラー処理

| 状況 | 動作 |
|---|---|
| コアからエラー値が返される（下の行の場合を除く） | `isError: true`、エラー内容の JSON を返す |
| `explain_cron` で `next_runs()` が `SchedulerError` を返す | `isError` を立てず、`next_runs: []` で返す |
| `count` が範囲外、または `from_time` の形式が不正 | `isError: true`、原因を含む JSON を返す |

---

## Testing Strategy

### 2 種類のテスト

- **例示ベースのテスト**: 具体的な例・境界値・エラーケースを検証する。
- **プロパティベーステスト（Hypothesis）**: 普遍的なプロパティを、生成した入力で検証する。

テストを書くときのルールは `.kiro/steering/testing-pbt.md` に従います。各プロパティテストの直前には、次の形式のコメントを書きます。

```python
# Feature: cronja, Property 1: Cron string round trip
# Validates: Requirements 2.11, 10.2
```

### Hypothesis ストラテジー（`tests/strategies.py`）

任意の部分集合だけで生成すると、全範囲・単一値・`*/n` といった形がほとんど出ず、定型の言い回しや 5・6 フィールドの分岐が検証されません。フィールドごとに形を混ぜて生成します。

```python
from hypothesis import strategies as st

def field_sets(lo: int, hi: int) -> st.SearchStrategy[frozenset[int]]:
    full = st.just(frozenset(range(lo, hi + 1)))
    single = st.integers(lo, hi).map(lambda v: frozenset({v}))
    contiguous = st.tuples(st.integers(lo, hi), st.integers(lo, hi)).map(
        lambda p: frozenset(range(min(p), max(p) + 1)))
    step = st.integers(2, hi - lo).map(lambda n: frozenset(range(lo, hi + 1, n)))
    arbitrary = st.frozensets(st.integers(lo, hi), min_size=1)
    return st.one_of(full, single, contiguous, step, arbitrary)

@st.composite
def cron_exprs(draw: st.DrawFn) -> CronExpr:
    return CronExpr(
        second=draw(st.one_of(st.just(frozenset({0})), field_sets(0, 59))),
        minute=draw(field_sets(0, 59)),
        hour=draw(field_sets(0, 23)),
        day=draw(field_sets(1, 31)),
        month=draw(field_sets(1, 12)),
        dow=draw(st.one_of(st.just(frozenset({1, 2, 3, 4, 5})),
                           st.just(frozenset({0, 6})), field_sets(0, 6))),
        year=draw(st.one_of(st.just(FULL_YEAR), field_sets(1970, 2099))),
    )

@st.composite
def hourly_cron_exprs(draw: st.DrawFn) -> CronExpr:
    # Property 7 用: 時・日・月・曜日・年は全範囲、秒と分だけ任意
    return CronExpr(
        second=draw(st.one_of(st.just(frozenset({0})), field_sets(0, 59))),
        minute=draw(field_sets(0, 59)),
        hour=FULL_HOUR, day=FULL_DAY, month=FULL_MONTH, dow=FULL_DOW, year=FULL_YEAR,
    )

bases = st.datetimes(min_value=datetime(1970, 1, 1), max_value=datetime(2099, 12, 31))
```

`CronExpr` は集合から直接組み立てます。文字列をパースして作ることはしません。

`tests/strategies.py` には、実装の `matches` に頼らない検証用の関数 `matches_oracle(x, dt)` も置きます。プロパティ 6 と 7 はこの関数で判定します。

### プロパティテストの概要

```python
# tests/test_roundtrip_properties.py

# Feature: cronja, Property 1: Cron string round trip
# Validates: Requirements 2.11, 10.2
@given(cron_exprs())
def test_cron_string_round_trip(x: CronExpr) -> None:
    assert parse(formatter.format(x)) == x

# Feature: cronja, Property 2: Japanese round trip
# Validates: Requirements 3.1, 3.9, 4.1, 10.3
@given(cron_exprs())
def test_japanese_round_trip(x: CronExpr) -> None:
    assert from_text(describe(x)) == x

# Feature: cronja, Property 3: Canonical form is idempotent
# Validates: Requirements 2.12, 10.4
@given(cron_exprs())
def test_canonical_form_idempotent(x: CronExpr) -> None:
    text = formatter.format(x)
    parsed = parse(text)
    assert isinstance(parsed, CronExpr)
    assert formatter.format(parsed) == text

# Feature: cronja, Property 4: Field count rule
# Validates: Requirements 2.1, 2.2, 2.3, 2.4, 10.5
@given(cron_exprs())
def test_field_count_rule(x: CronExpr) -> None:
    count = len(formatter.format(x).split())
    if x.year != FULL_YEAR:
        assert count == 7
    elif x.second != ZERO_SECOND:
        assert count == 6
    else:
        assert count == 5
```

```python
# tests/test_schedule_properties.py

# Feature: cronja, Property 5: Run times are after base, strictly ascending, at most n
# Validates: Requirements 5.1, 5.2, 5.3, 10.6
@given(cron_exprs(), bases, st.integers(1, 20))
def test_runs_are_chronological(x: CronExpr, base: datetime, n: int) -> None:
    result = next_runs(x, base, n)
    if isinstance(result, list):
        assert len(result) <= n
        assert all(t > base for t in result)
        assert all(a < b for a, b in zip(result, result[1:]))

# Feature: cronja, Property 6: Every run time matches the expression
# Validates: Requirements 5.4, 5.5, 5.6, 10.7
@given(cron_exprs(), bases, st.integers(1, 20))
def test_runs_match_expression(x: CronExpr, base: datetime, n: int) -> None:
    result = next_runs(x, base, n)
    if isinstance(result, list):
        assert all(matches_oracle(x, t) for t in result)

# Feature: cronja, Property 7: No run time is skipped
# Validates: Requirements 5.7, 10.8
@given(hourly_cron_exprs(), bases, st.integers(1, 5))
def test_no_run_is_skipped(x: CronExpr, base: datetime, n: int) -> None:
    result = next_runs(x, base, n)
    if isinstance(result, list):
        points = [base.replace(microsecond=0)] + result
        for start, end in zip(points, points[1:]):
            t = start + timedelta(seconds=1)
            while t < end:
                assert not matches_oracle(x, t)
                t += timedelta(seconds=1)
```

```python
# tests/test_robustness_properties.py

# Feature: cronja, Property 8: parse and from_text never raise
# Validates: Requirements 1.15, 4.5, 10.9
@given(st.text())
def test_parse_never_raises(s: str) -> None:
    assert isinstance(parse(s), (CronExpr, ParseError))

@given(st.text())
def test_from_text_never_raises(s: str) -> None:
    assert isinstance(from_text(s), (CronExpr, ParseError))
```

### 例示ベースのテストの方針

プロパティテストと重複しない範囲に絞ります。

- `describe()`: 上の「出力例」の表の全行
- `from_text()`: 表記ゆれ（「9時」「9:00」「午前9時」「午後9時」「月〜金」「週末」「毎週月曜日」、全角数字）とエラーケース
- `parse()`: 各エラー条件（フィールド数、範囲外、`a > b`、ステップ値、不正な文字）
- `next_runs()`: 日と曜日の両方指定、月末・年末・うるう年のまたぎ、2099 年末の打ち切り、`n < 1`
- `lint()`: 各警告を単独で出す式、複数の警告が同時に出る式、警告が出ない式（`0 0 31 * *`、`0 0 1,15 * *`、`0 0 1,16 * *` を含む）
- `format()`: 2 要素の集合の扱い（分 `{0, 30}` → `*/30`、分 `{0, 45}` → `0,45`、曜日 `{0, 6}` → `0,6`）
- `cli.py`: 各サブコマンドの出力と終了コード
