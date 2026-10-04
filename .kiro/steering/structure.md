# プロジェクト構成

```
cronja/
├── .kiro/                   # specs、steering、hooks、settings、agents
├── plugin.json              # Kiro Power のマニフェスト（リポジトリのルートが Power）
├── mcp.json                 # Power に同梱する MCP 設定
├── skills/                  # Power に同梱する Agent Skills
├── src/
│   └── cronja/
│       ├── __init__.py      # コアの公開 API を再エクスポート
│       ├── model.py         # ドメインの型と、各フィールドの値の範囲
│       ├── parser.py        # cron 文字列のパース
│       ├── formatter.py     # 正規形の cron 文字列の生成
│       ├── describe.py      # 日本語の説明文の生成
│       ├── from_text.py     # 日本語の文のパース
│       ├── schedule.py      # 次回実行時刻の計算
│       ├── lint.py          # 書き間違いの警告
│       ├── cli.py           # CLI
│       └── mcp_server.py    # MCP サーバー
├── tests/
│   ├── strategies.py        # 共通の Hypothesis ストラテジー
│   ├── test_*_properties.py # プロパティベーステスト
│   └── test_*.py            # 例示ベースのテスト
├── pyproject.toml
├── README.md
└── LICENSE
```

## 層の分け方

```
cli.py、mcp_server.py   →   コア（parser、formatter、describe、from_text、schedule、lint）   →   model.py
```

- **コアのモジュール**は、`model.py` の型を扱う純粋関数だけで構成する。入出力をせず、グローバルな状態を持たない。`cli`、`mcp_server`、`argparse`、`mcp` を import せず、print もしない。
- **`cli.py` と `mcp_server.py`** は薄いアダプタにする。引数を受け取り、コアを呼び、結果を整形するだけで、変換のロジックは置かない。同じ入力に対して、両者は同じ情報を返すこと。
- `describe.py` と `from_text.py` は、1つの文法の両方向を実装している。片方を変更したら、もう片方と、`design.md` に書かれた文法を必ず確認する。

## 規約

- ソースは `src/cronja/` に置く（src レイアウト）。1モジュール1責務とし、上の一覧のとおりに分ける。統合や名前の変更をするときは、このファイルと、これらのパスを条件にしているフックも合わせて更新する。
- `.kiro/` はコミットする。`.gitignore` に入れない。
