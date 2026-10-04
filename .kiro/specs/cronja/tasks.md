# Implementation Plan: cronja

## Overview

最初にプロジェクト雛形を作り、コア層（副作用のない純粋関数群）を依存関係の順に実装する。各モジュールの直後にプロパティベーステストを配置する。その後、例示ベースのテスト・CLI・MCP サーバー・README の順に進める。コア層は標準ライブラリのみで実装し、外部の cron ライブラリは使用しない。

---

## Tasks

- [ ] 1. プロジェクト雛形と `pyproject.toml` のセットアップ
  - `src/cronja/__init__.py` を含む `src` レイアウトのディレクトリ構造を作成する
  - `pyproject.toml` を作成する（ビルドバックエンド: `hatchling`、`requires-python = ">=3.12"`）
  - `[project.scripts]` に `cronja = "cronja.cli:main"` と `cronja-mcp = "cronja.mcp_server:main"` を定義する
  - `[project.dependencies]` に `mcp` のみを設定する
  - `[dependency-groups]` の `dev` に `pytest`、`hypothesis`、`ruff` を設定する
  - `tests/` ディレクトリと空の `tests/__init__.py` を作成する
  - `[tool.pytest.ini_options]` に `testpaths = ["tests"]` を設定する
  - `uv sync` が成功することを確認する
  - _Requirements: 9.1, 9.2, 9.4_

- [ ] 2. `model.py` — データモデルとフィールド定数の実装
  - [ ] 2.1 `CronExpr`、`ParseError`、`SchedulerError`、`LintWarning` を `@dataclass(frozen=True)` で定義する
    - `CronExpr` のフィールドはすべて `frozenset[int]`（`second`, `minute`, `hour`, `day`, `month`, `dow`, `year`）
    - `ParseError`: `field: str | None`、`value: str`、`reason: str`
    - `SchedulerError`: `reason: str`
    - `LintWarning`: `code: str`、`message: str`
    - すべてのフィールドに型ヒントを付ける
    - _Requirements: 9.5, 9.6_
  - [ ] 2.2 `FIELD_RANGES` 辞書と `FULL_*`・`ZERO_SECOND` 定数を定義する
    - `FIELD_RANGES: dict[str, range]` に `second`（0〜59）、`minute`（0〜59）、`hour`（0〜23）、`day`（1〜31）、`month`（1〜12）、`dow`（0〜6）、`year`（1970〜2099）を設定する
    - `FULL_SECOND`、`FULL_MINUTE`、`FULL_HOUR`、`FULL_DAY`、`FULL_MONTH`、`FULL_DOW`、`FULL_YEAR`、`ZERO_SECOND` を定義する
    - _Requirements: 9.5, 9.6_
  - [ ] 2.3 `step_of(values: frozenset[int], field: str) -> int | None` を実装する
    - `values` がステップ集合（`*/n` と等価）ならステップ値 `n` を返し、そうでなければ `None` を返す
    - 判定条件: `frozenset(range(rng.start, rng.stop, n))` と一致し、かつ「要素数 ≥ 3」または「`len(rng) % n == 0`」を満たす
    - `formatter`・`describe`・`lint` の 3 か所から共有される
    - _Requirements: 2.8, 6.2_

- [ ] 3. `parser.py` — `parse(s: str) -> CronExpr | ParseError` の実装
  - [ ] 3.1 フィールド分割とフィールド数の検証を実装する
    - `s.split()` でフィールドに分割し、5・6・7 フィールド以外は `ParseError` を返す
    - フィールド数に応じて、秒（`{0}` で補完）と年（全範囲で補完）を決定する
    - `[0-9*/,-]` 以外の文字を含むフィールドは `ParseError` を返す（英字名、`L`・`W`・`#`・`?`、全角数字などを拒否）
    - _Requirements: 1.1, 1.2, 1.3, 1.4, 1.12_
  - [ ] 3.2 各フィールドの展開ロジックを実装する（`*`、整数、`a-b`、`*/n`、`a-b/n`、カンマ区切り）
    - `*/n` → `range(min, max+1, n)` の集合
    - `a-b/n` → `range(a, b+1, n)` の集合
    - `a-b` → `range(a, b+1)` の集合
    - `*` → 有効範囲の全整数の集合
    - 整数 → `{v}`
    - カンマ区切りは上記の和集合
    - _Requirements: 1.5, 1.6, 1.7, 1.8, 1.9_
  - [ ] 3.3 明示的なバリデーションと曜日の正規化を実装する
    - 値が有効範囲外の場合（曜日は 0〜7 を許す）は `ParseError` を返す
    - `a > b` の場合は `ParseError` を返す
    - ステップ値 `n < 1` または `n > 最大値 - 最小値` の場合は `ParseError` を返す
    - 曜日の 7 を 0 に正規化する
    - 最外層の `try/except Exception` を最後の受け皿としてのみ配置する
    - _Requirements: 1.10, 1.11, 1.13, 1.14, 1.15, 1.16_

- [ ] 4. `formatter.py` — `format(x: CronExpr) -> str` の実装
  - [ ] 4.1 `_format_field` を実装する（優先順位順）
    - 優先順位: `*`（全範囲）→ 単一値 → `a-b`（連続範囲）→ `*/n`（ステップ集合）→ `a-b/n`（3 要素以上の等差数列、n≥2）→ カンマ区切りリスト
    - 2 要素の集合は `*/n` でも `a-b/n` でもなく、カンマ区切りで出力する
    - _Requirements: 2.5, 2.6, 2.7, 2.8, 2.9, 2.10_
  - [ ] 4.2 フィールド数の決定ロジックと `format` の本体を実装する
    - 秒が `{0}` かつ年が全範囲 → 5 フィールド（分 時 日 月 曜日）
    - 秒が `{0}` 以外かつ年が全範囲 → 6 フィールド（秒 分 時 日 月 曜日）
    - 年が全範囲でない → 7 フィールド（秒 分 時 日 月 曜日 年）
    - _Requirements: 2.1, 2.2, 2.3, 2.4_

  - [ ] 4.3 プロパティテスト — `tests/strategies.py` と `tests/test_roundtrip_properties.py` を実装する（Property 1, 3, 4）
    - テストの書き方とコメントの形式は `.kiro/steering/testing-pbt.md` に従う（各テストの直前に `# Feature: cronja, Property <N>: <英語の名前>` と `# Validates: Requirements <X.Y>` の 2 行を書く）
    - `CronExpr` は集合から直接組み立てる。文字列をパースして作らない
    - **`tests/strategies.py`** を作成する（後続のすべてのプロパティテストが共有する）
      - `field_sets(lo, hi)`: 全範囲・単一値・連続範囲・ステップ・任意部分集合を混合して生成する `SearchStrategy[frozenset[int]]`
      - `cron_exprs()`: 7 フィールドの `CronExpr` を生成する複合ストラテジー
      - `hourly_cron_exprs()`: Property 7 用（時・日・月・曜日・年は全範囲、秒と分だけ任意）
      - `bases`: `datetime(1970,1,1)` 〜 `datetime(2099,12,31)` の `SearchStrategy[datetime]`
      - `matches_oracle(x, dt) -> bool`: 実装の `matches` に頼らない参照実装（Property 6, 7 の検証用）
    - **Property 1**: `parse(format(x)) == x`（cron 文字列のラウンドトリップ）
      - **Validates: Requirements 2.11, 10.2**
    - **Property 3**: `format(parse(format(x))) == format(x)`（正規形の冪等性）
      - **Validates: Requirements 2.12, 10.4**
    - **Property 4**: `format(x)` のフィールド数が秒・年の規則に従う
      - **Validates: Requirements 2.1, 2.2, 2.3, 2.4, 10.5**

- [ ] 5. `describe.py` — `describe(x: CronExpr) -> str` の実装
  - [ ] 5.1 列挙ヘルパー（`_int_enum`、`_dow_enum`）を実装する
    - 値を昇順に並べ、3 つ以上連続する値は `a〜b`、2 つ以下は `・` 区切りで出力する
    - `_dow_enum` は 0=日、1=月、…、6=土 の名前を使う
    - _Requirements: 3.9_
  - [ ] 5.2 `_describe_date(x)` を実装する（日付の部分）
    - 6 行の判定表を上から順に評価し、最初に該当した言い回しを返す（毎日, 平日, 土日, 毎週◯曜, 毎月◯日, フォールバック）
    - 月が制限されている場合の 6b〜6e のサブケースを実装する
    - _Requirements: 3.2, 3.3, 3.4_
  - [ ] 5.3 `_describe_time(x)` を実装する（時刻の部分）
    - 7 行の判定表を上から順に評価し、種別（`repeat`・`exact`・`regular`）とともに文字列を返す
    - `step_of` を使って「n秒おき」「n分おき」を判定する
    - `time_regular` は BNF の `<hour_part><minute_part><second_part>` に従い出力する
    - _Requirements: 3.5_
  - [ ] 5.4 `_describe_year(x)` と `describe(x)` の組み合わせロジックを実装する
    - 年が全範囲 → 空文字列、制限あり → `（対象年: <int_enum>）`
    - 組み合わせ規則: 日付が「毎日」かつ時刻が `repeat` → 時刻のみ、日付が「毎日」かつ時刻が `exact` → `日付 + 時刻`（「の」なし）、それ以外 → `日付 + の + 時刻`
    - 要件 3.8 の出力例テーブルの全行を満たすこと
    - _Requirements: 3.1, 3.2, 3.6, 3.7, 3.8, 3.9, 3.10_

- [ ] 6. `from_text.py` — `from_text(s: str) -> CronExpr | ParseError` の実装
  - [ ] 6.1 前処理と年サフィックスの解析を実装する
    - `unicodedata.normalize("NFKC", s)` で正規化し、空白を取り除く
    - 末尾の `(対象年:<int_enum>)` を切り離して年の集合に変換する（なければ全範囲）
    - 空文字列になったら `ParseError` を返す
    - _Requirements: 4.4, 4.5_
  - [ ] 6.2 `_parse_time(s)` ヘルパーを実装する
    - パターン順: `毎秒`, `n秒おき`, `毎分`, `n分おき`, `<hour_part><minute_part><second_part>`, `h時`, `h:mm`, `午前/午後h時[m分]`
    - 分が書かれていない場合は 0 に補完する
    - 範囲区切りは `〜`（U+301C）と `~` の両方を受け付ける
    - 列挙の区切りは `・`、`,`、`、` を受け付ける（日付の部分の列挙も同じ）
    - _Requirements: 4.2, 4.7_
  - [ ] 6.3 `_parse_date(s)` ヘルパーを実装する
    - パターン順: `毎日`, `平日`, `土日`/`週末`, `毎月◯日または◯曜`, `毎月◯日`, `<int_enum>月の<month_body>`, `[毎週]<dow_enum>[曜|曜日]`
    - `月〜金` のような範囲指定を受け付ける（要件 4.3）
    - _Requirements: 4.2, 4.3_
  - [ ] 6.4 本体の解析ロジックを実装する
    - 試行順: 時刻のみ（日付は「毎日」）→ `毎日` + 時刻 → 最後の `の` で分割 → どれも不一致なら `ParseError`
    - 展開した値が有効範囲外なら `ParseError`
    - 最外層の `try/except Exception` を最後の受け皿としてのみ配置する
    - _Requirements: 4.1, 4.4, 4.5, 4.6, 4.8_

  - [ ] 6.5 プロパティテスト — `tests/test_roundtrip_properties.py` に Property 2 を追加、`tests/test_robustness_properties.py` を作成（Property 8）
    - **Property 2**: `from_text(describe(x)) == x`（日本語のラウンドトリップ）
      - **Validates: Requirements 3.1, 3.9, 4.1, 10.3**
    - **Property 8 — `test_parse_never_raises`**: 任意の文字列 `s` に対して `parse(s)` が例外なく `CronExpr | ParseError` を返す
      - **Validates: Requirements 1.15, 10.9**
    - **Property 8 — `test_from_text_never_raises`**: 任意の文字列 `s` に対して `from_text(s)` が例外なく `CronExpr | ParseError` を返す
      - **Validates: Requirements 4.5, 10.9**

- [ ] 7. `schedule.py` — `next_runs`・`matches` の実装
  - [ ] 7.1 `_day_matches(x, dt) -> bool` と `matches(x, dt) -> bool` を実装する
    - `dow` 変換: `(dt.weekday() + 1) % 7`
    - 日と曜日の OR 規則: 両方制限 → どちらかに一致すれば真、片方のみ制限 → その側のみ判定
    - `matches` は `dt.microsecond == 0` も検査する
    - _Requirements: 5.4, 5.5, 5.6_
  - [ ] 7.2 スキップアヘッドアルゴリズムで `next_runs(x, base, n) -> list[datetime] | SchedulerError` を実装する
    - `n < 1` → `SchedulerError`
    - `base` の 1 秒後から開始し、マイクロ秒は切り捨てる
    - 年 → 月 → 日 → 時 → 分 → 秒 の順に一致しない単位をスキップする
    - 不一致時は丸ごとスキップしてループ先頭（年の判定）に戻る
    - 探索上限: `t.year > 2099` で打ち切る
    - 結果が 0 件なら `SchedulerError` を返す、`n` 件未満でも見つかった分だけ返す
    - _Requirements: 5.1, 5.2, 5.3, 5.5, 5.6, 5.7, 5.8, 5.9, 5.10, 5.11_

  - [ ] 7.3 プロパティテスト — `tests/test_schedule_properties.py` を作成（Property 5, 6, 7）
    - **Property 5**: `next_runs` の結果が `base` より後で、厳密に昇順で、重複なく、件数が `n` 以下
      - **Validates: Requirements 5.1, 5.2, 5.3, 10.6**
    - **Property 6**: `next_runs` の各結果が `x` の全フィールド条件を満たす（`matches_oracle` で検証）
      - **Validates: Requirements 5.4, 5.5, 5.6, 10.7**
    - **Property 7**: `base` と最初の結果の間、および `next_runs` の連続する 2 結果の間に、`x` にマッチする時刻が存在しない（`hourly_cron_exprs` を使用し、`matches_oracle` で 1 秒ずつ検証する）
      - **Validates: Requirements 5.7, 10.8**

- [ ] 8. `lint.py` — `lint(x, base) -> list[LintWarning]` の実装
  - [ ] 8.1 補助関数 `_is_leap(year: int) -> bool` と有効な月日の組み合わせ判定を実装する
    - うるう年判定: `(year % 4 == 0 and year % 100 != 0) or year % 400 == 0`
    - 有効な月日: 月 `m` と日 `d` の組のうち `d <= m 月の日数`（2 月は 29 日まで）
    - _Requirements: 6.3, 6.4_
  - [ ] 8.2 7 種類の警告条件をすべて実装する
    - `OR_CONDITION`: 日と曜日の両方が制限されている
    - `UNEVEN_STEP`: 秒・分・時・日・月・曜日のいずれかがステップ集合で `値の個数 % n ≠ 0`（年は対象外）
    - `NO_VALID_DATE`: 曜日が全範囲で、有効な月日の組み合わせが 0 件、またはうるう年なしで 2/29 のみ
    - `LEAP_YEAR_ONLY`: 曜日が全範囲で、有効な組み合わせが 2/29 のみで、かつ年にうるう年が含まれる
    - `ALL_YEARS_PAST`: 年が制限されていて、年の最大値 < `base.year`
    - `EVERY_SECOND`: 7 フィールドすべてが全範囲
    - `EVERY_MINUTE`: 秒が `{0}` で、分・時・日・月・曜日がすべて全範囲
    - 複数条件が成立する場合はすべての警告をリストで返す、0 件の場合は空リストを返す
    - _Requirements: 6.1, 6.2, 6.3, 6.4, 6.5, 6.6, 6.7, 6.8, 6.9_
  - [ ] 8.3 `src/cronja/__init__.py` でコアの公開 API を再エクスポートする
    - `CronExpr`、`ParseError`、`SchedulerError`、`LintWarning`、`parse`、`describe`、`from_text`、`next_runs`、`lint` と、`formatter` モジュールを公開する
    - `__all__` を定義する。`cli` と `mcp_server` は import しない
    - _Requirements: 9.3_

- [ ] 9. チェックポイント — コア層の動作確認
  - すべてのテストが通ることを確認する。問題があればユーザーに確認する。
  - `uv run ruff check .` と `uv run ruff format .` を実行して lint とフォーマットを通す。

- [ ] 10. ユニットテスト — コアモジュールの例示ベーステスト
  - [ ] 10.1 `tests/test_parser.py` を作成する
    - エラー条件（フィールド数、範囲外、`a > b`、ステップ値、不正な文字）
    - 5・6・7 フィールドの補完、曜日 7→0 正規化
    - _Requirements: 1.1-1.16_
  - [ ]* 10.2 `tests/test_formatter.py` を作成する
    - 2 要素の集合の扱い: 分 `{0,30}` → `*/30`、分 `{0,45}` → `0,45`、曜日 `{0,6}` → `0,6`
    - 5・6・7 フィールドの切り替え
    - _Requirements: 2.1-2.12_
  - [ ]* 10.3 `tests/test_describe.py` を作成する
    - 要件 3.8 の出力例テーブルの全行
    - `0 9-17 * * *`、`5,35 * * * *`、`0 9 13 * 5`（OR）など
    - _Requirements: 3.1-3.10_
  - [ ]* 10.4 `tests/test_from_text.py` を作成する
    - 表記ゆれ: 「9時」「9:00」「午前9時」「午後9時」「月〜金」「週末」「毎週月曜日」、全角数字
    - エラーケース: 空文字列、空白のみ、スケジュール以外の文字列
    - _Requirements: 4.1-4.8_
  - [ ]* 10.5 `tests/test_schedule.py` を作成する
    - 日と曜日の両方指定（OR 規則）
    - 月末・年末・うるう年のまたぎ
    - 2099 年末の打ち切り、`n < 1` のエラー
    - _Requirements: 5.1-5.11_
  - [ ]* 10.6 `tests/test_lint.py` を作成する
    - 各警告を単独で出す式、複数の警告が同時に出る式
    - 警告が出ない式（`0 0 31 * *`、`0 0 1,15 * *`、`0 0 1,16 * *` を含む）
    - _Requirements: 6.1-6.9_

- [ ] 11. `cli.py` — CLI インターフェースの実装
  - [ ] 11.1 `argparse` でサブコマンド構造（`explain`・`make`・`next`）を構築する
    - `explain "<cron式>" [--from <ISO日時>]`
    - `make "<日本語>"`
    - `next "<cron式>" [--count N] [--from <ISO日時>]`
    - `main(argv: list[str] | None = None) -> int` の形にする。終了コードを戻り値で返し、`sys.exit` はエントリポイントの呼び出し側に任せる（テストから `main([...])` で呼べるようにするため）
    - 基準時刻は `cli.py` で決める（`--from` があればその値、なければ `datetime.now()` を秒単位に切り捨てた値）
    - `main` の中で標準入力・標準出力・標準エラーを UTF-8 に固定する。`reconfigure` を持たないストリームでは何もしない
    - _Requirements: 7.1-7.11_
  - [ ] 11.2 各サブコマンドのハンドラを実装する
    - `explain`: `parse` → `describe` + `next_runs`（3 件）+ `lint` の結果をまとめて表示；`next_runs` が `SchedulerError` の場合は終了コード 0 で「実行予定はありません」を表示
    - `make`: `from_text` → cron 文字列を標準出力に表示
    - `next`: `parse` → `next_runs`（`--count` 件、デフォルト 1）を 1 行 1 件で表示
    - コアからエラー値が返された場合（`explain` の `SchedulerError` を除く）は stderr に表示して終了コード 1
    - `--from` の形式検査（`YYYY-MM-DDTHH:MM:SS`）、`--count` の範囲検査（1〜100）は `cli.py` で行い、終了コード 1 にする
    - _Requirements: 7.1-7.11_

  - [ ]* 11.3 `tests/test_cli.py` を作成する
    - 各サブコマンドの出力内容と終了コードを検証する
    - エラー時（不正な cron 式、不正な `--from`、範囲外 `--count`）のテスト
    - `explain` で `SchedulerError` が返る場合のテスト
    - _Requirements: 7.1-7.11_

- [ ] 12. `mcp_server.py` — MCP サーバーインターフェースの実装
  - [ ] 12.1 `mcp` SDK を使って stdio トランスポートのサーバーを構成し、`explain_cron`・`make_cron`・`next_runs` の 3 ツールを登録する
    - 各ツールに空でない説明文と入力スキーマを定義する
    - 起動時に標準入出力を UTF-8 に固定する
    - _Requirements: 8.1, 8.6, 8.7_
  - [ ] 12.2 各ツールのハンドラを実装する
    - `explain_cron`（`cron_expression` 必須、`from_time` 省略可）: 説明文・次の 3 件の実行時刻・lint 警告を構造化データで返す；`SchedulerError` の場合は `isError` なし・`next_runs: []`
    - `make_cron`（`text` 必須）: `{"cron_expression": "..."}` を返す
    - `next_runs`（`cron_expression` 必須、`count` 省略可・デフォルト 1・1〜100、`from_time` 省略可）: `{"runs": [...]}` を返す
    - コアからエラー値が返された場合は `isError: true` とエラー内容の JSON を返す
    - `count` の範囲外・`from_time` の形式不正は `isError: true` で返す
    - 各ツールの処理は、通信を介さずに呼べる通常の関数として実装し、MCP の登録はその関数を包むだけにする
    - エントリポイントは `main() -> None`。標準入力・標準出力・標準エラーを UTF-8 に固定する
    - _Requirements: 8.1-8.10_
  - [ ] 12.3 `tests/test_mcp_server.py` を作成する
    - 通信を介さず、各ツールの処理関数を直接呼んで検証する
    - 3 ツールの成功時の出力の形（`description`・`next_runs`・`warnings`、`cron_expression`、`runs`）
    - エラー時の出力（不正な cron 式、読めない日本語、範囲外の `count`、不正な `from_time`）
    - `explain_cron` で実行時刻が見つからない場合に、エラーにならず `next_runs` が空になること
    - 登録されているツールが 3 つで、それぞれ説明文が空でないこと
    - _Requirements: 8.1-8.10_

- [ ] 13. チェックポイント — CLI・MCP サーバーの動作確認
  - すべてのテストが通ることを確認する。問題があればユーザーに確認する。
  - `uv run ruff check .` と `uv run ruff format .` を実行して lint とフォーマットを通す。

- [ ] 14. `README.md` の作成
  - 英語で記述する
  - 概要セクション（2〜3 文でツールの目的・対象ユーザー・主要機能を説明）
  - クイックスタートセクション（前提条件（uv のインストール）、インストールコマンド、各サブコマンドの実行例と期待される出力例）
  - 「Kiro features used」の表（7 行: Specs, Steering, Hooks, Property-based testing, Powers, MCP, Custom agents）を含め、各行に 1 文以内のユースケース説明とファイルパスを記載する（未確定の行は TODO）
  - リポジトリのルートに `LICENSE`（MIT、著作権者は `noooomgs`、年は 2026）を置き、`pyproject.toml` の `license` と一致させる
  - _Requirements: 11.1, 11.2, 11.3, 11.4_

- [ ] 15. 最終チェックポイント — 全テスト通過の確認
  - すべてのテストが通ることを確認する。問題があればユーザーに確認する。
  - `uv run ruff check .` と `uv run ruff format .` を実行して lint とフォーマットを通す。

---

## Notes

- `*` を付けたサブタスクはオプションで、MVP を優先する場合はスキップできる
- プロパティテストのタスク（4.3、6.5、7.3）は必須。スキップしない
- 各タスクは対応する要件番号で追跡できる
- チェックポイントで段階的に動作確認を行う
- プロパティテストは各実装モジュールの直後に配置し、早期にバグを検出する
- `format` は組み込み関数と同名のため、ほかのモジュールからは `from cronja import formatter; formatter.format(x)` の形で呼ぶ

## Task Dependency Graph

同じファイルを編集するタスクは、同じ wave に入れない（2.x は `model.py`、5.x は `describe.py`、6.x は `from_text.py`）。

```json
{
  "waves": [
    { "id": 0, "tasks": ["1"] },
    { "id": 1, "tasks": ["2.1"] },
    { "id": 2, "tasks": ["2.2"] },
    { "id": 3, "tasks": ["2.3"] },
    { "id": 4, "tasks": ["3.1"] },
    { "id": 5, "tasks": ["3.2"] },
    { "id": 6, "tasks": ["3.3"] },
    { "id": 7, "tasks": ["4.1"] },
    { "id": 8, "tasks": ["4.2"] },
    { "id": 9, "tasks": ["4.3"] },
    { "id": 10, "tasks": ["5.1"] },
    { "id": 11, "tasks": ["5.2"] },
    { "id": 12, "tasks": ["5.3"] },
    { "id": 13, "tasks": ["5.4"] },
    { "id": 14, "tasks": ["6.1"] },
    { "id": 15, "tasks": ["6.2"] },
    { "id": 16, "tasks": ["6.3"] },
    { "id": 17, "tasks": ["6.4"] },
    { "id": 18, "tasks": ["6.5"] },
    { "id": 19, "tasks": ["7.1"] },
    { "id": 20, "tasks": ["7.2"] },
    { "id": 21, "tasks": ["7.3"] },
    { "id": 22, "tasks": ["8.1"] },
    { "id": 23, "tasks": ["8.2"] },
    { "id": 24, "tasks": ["8.3"] },
    { "id": 25, "tasks": ["10.1", "10.2", "10.3", "10.4", "10.5", "10.6"] },
    { "id": 26, "tasks": ["11.1"] },
    { "id": 27, "tasks": ["11.2"] },
    { "id": 28, "tasks": ["11.3"] },
    { "id": 29, "tasks": ["12.1"] },
    { "id": 30, "tasks": ["12.2"] },
    { "id": 31, "tasks": ["12.3"] },
    { "id": 32, "tasks": ["14"] }
  ]
}
```
