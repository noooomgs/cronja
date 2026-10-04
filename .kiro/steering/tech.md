# 技術スタック

## 言語とツール

| ツール | 用途 |
|------|---------|
| Python 3.12 | 言語。新しい構文を使う（`X \| Y` の union、`match`、frozen な dataclass）。 |
| `uv` | 依存関係の管理、仮想環境、コマンド実行。`pip` や `python -m venv` を直接使わない。 |
| `pytest` | テスト実行 |
| `hypothesis` | プロパティベーステスト |
| `mcp` | 公式の MCP Python SDK。MCP サーバーのモジュールだけが使う。 |
| `argparse` | CLI（標準ライブラリ） |
| `ruff` | lint とフォーマット。`black` や `flake8` を追加しない。 |
| `hatchling` | `pyproject.toml` のビルドバックエンド |

## 依存関係のルール

- コア（MCP サーバー以外のすべて）は**標準ライブラリだけ**で実装する。
- 外部の cron ライブラリ（`croniter`、`cron-descriptor`、`cronsim` など）を**追加しない**。パース、説明文の生成、実行時刻の計算は、このリポジトリ内で実装する。
- 実行時の依存は `mcp` のみ。開発時の依存は `pytest`、`hypothesis`、`ruff`。
- 依存を追加するのは、spec のタスクが求めたときだけにする。

## エントリポイント

`pyproject.toml` の `[project.scripts]` に定義する。

- `cronja` → CLI
- `cronja-mcp` → MCP サーバー（stdio）

どちらも、リポジトリ内での `uv run <名前>` と、任意の場所からの `uvx --from git+<リポジトリURL> <名前>` の両方で動くこと。

## よく使うコマンド

```bash
uv sync                        # 依存関係をインストール
uv run pytest -q               # 全テストを実行
uv run pytest -q -x            # 最初の失敗で止める（保存時のフックが使う）
uv run ruff check .            # lint
uv run ruff format .           # フォーマット
```

タスクを完了とする前に、`uv run ruff check .` と `uv run ruff format .` を実行する。

## コーディングのルール

- すべての関数と dataclass のフィールドに型ヒントを付ける。
- コアは時計を読まない。現在時刻は必ず引数（`base`）で受け取る。
- datetime は naive なものを使う。タイムゾーンは扱わない。
- エラーは例外で投げず、**戻り値で返す**。失敗しうる関数は、結果の型とエラーの型の union を返し、呼び出し側は `isinstance` か `match` で分岐する。例外はプログラムの誤りに対してだけ使う。
- ドメインの型は不変にする（`@dataclass(frozen=True)`、フィールドの値は `frozenset[int]`）。

## 実行環境についての注意

- コードは Windows、Linux、macOS のどれでも動くこと（Power はどの OS でも `uvx` 経由で起動される）。
- このツールは日本語を出力する。CLI と MCP サーバーは、起動時に標準入力・標準出力・標準エラーを UTF-8 に固定する（例: `sys.stdout.reconfigure(encoding="utf-8")`）。環境によっては既定の文字コードが UTF-8 でないため。
- コミットする設定ファイル（`.kiro/settings/mcp.json`、`mcp.json`、フック）に、絶対パスや特定のマシンに依存するパスを書かない。
