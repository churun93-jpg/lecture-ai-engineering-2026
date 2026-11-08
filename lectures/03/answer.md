# 第3回 解答

## 演習

- 第1回（素のLLM）とエージェントの回答の違い：
  - 第1回の素のLLM（`app.ask`）は講座固有の情報を持たないため、「宿題の3レベル」について一般論から「レベル1（必須要件）〜レベル3（加点要素）」などと推測・断定（ハルシネーション）して回答していた。
  - 第3回のエージェント（`app.agent`）は、回答前に `search_glossary` ツールを呼び出して `data/glossary.json` を検索するため、「基礎（講義内容の確認）」「実装（自分でコードを書く）」「発展（調査や試行）」の3段階であり、どれか1つに取り組めば完了・原則採点しないという正確な内容を回答し、末尾に関係する回（第1回）も示した。
- `--trace` で見えたツール呼び出し（ツール名と引数）：
  - 「LoRAとQLoRAの違いは？」実行時：
    - `search_glossary({'query': 'LoRA'})`
    - `search_glossary({'query': 'QLoRA'})`
  - 「第2回の内容から確認問題を1問出して」実行時：
    - `make_quiz({'lecture': 2})`
- 1つの質問での LLM 呼び出し回数（`requests`）：
  - `requests=2`（1回目のLLM呼び出しでツール呼び出しを判断・依頼し、プログラムがツールを実行した結果を渡して2回目のLLM呼び出しで最終回答を生成したため）
- 追加したツールと、docstring に書いた説明：
  - 追加したツール：`make_quiz(lecture: int) -> str`
  - docstring に書いた説明：
    `指定した回（1始まりの整数）の用語から1つを選び、確認問題と答えを返す。「第2回の確認問題を出して」のように、復習用の問題を求められたときに使う。`

## 宿題

- 取り組んだレベル：基礎
- やったこと：
  - `app/tools.py` の `search_glossary` 関数の docstring を以下のように書き換え、同じ質問 `"LoRAとQLoRAの違いは？"` を `--trace` 付きで実行して、ツールの呼ばれ方・回数・引数および回答がどう変わるかを比較した。
  - **書き換え前**：`query には調べたい用語を1つだけ入れる（例: "LoRA"）。`
  - **書き換え後**：`複数の用語を調べるときも、query には調べたい用語をまとめて1つの文字列として入れ、1回だけ呼び出すこと（例: "LoRA QLoRA"）。`
- 結果：
  - **書き換え前の `--trace`**：
    `search_glossary({'query': 'LoRA'})` と `search_glossary({'query': 'QLoRA'})` のように1語ずつ別々にツールが呼ばれ、`tool_calls=2`、`requests=2` となった。
  - **書き換え後の `--trace`**：
    ```text
    [tool call]   search_glossary({'query': 'LoRA QLoRA'})
    [tool return] search_glossary -> [{"term": "LoRA", "lecture": 2, "definition": "事前学習済みの重みは固定したまま、重みの更新分を2つの小さな行列の積（低ランク行列）で表し、その小さな行列だけを学習するPEFT手法。学習するパラ…
    [usage] RunUsage(cost=Decimal('0.0008173'), details={'text_prompt_tokens': 1091}, input_text_tokens=1091, input_tokens=1091, output_tokens=196, requests=2, tool_calls=1)
    ```
    docstring の指示通り、引数が `{'query': 'LoRA QLoRA'}` と1つの文字列にまとめられ、ツール呼び出し回数が `tool_calls=2` から `tool_calls=1` に変化した（`requests=2` は同じ）。
  - **回答の変化**：
    `search_glossary` の内部実装に部分一致の判定（`len(n) >= 3 and n in q`）が含まれているため、`'LoRA QLoRA'` という1回のクエリでも `LoRA` と `QLoRA` の両方のエントリが返り、回答本文も変更前と同様に用語集の定義に沿った正確な比較説明と関係する回 `（第2回）` が出力された。
- 気づき・疑問：
  - 関数の処理（Pythonのロジック）を一切変更しなくても、docstring（自然言語による説明文）を1文書き換えるだけで、LLMがツールに渡す引数の形式や呼び出し回数（`tool_calls: 2 → 1`）を直接コントロールできることを体験できた。
  - 今回は `search_glossary` の実装が `n in q`（3文字以上の用語名がクエリ文字列に含まれるか）も判定する仕組みだったため `'LoRA QLoRA'` の一括検索でも両方ヒットしたが、もし2文字以下の短い用語（`Pi` や `r` など）をまとめて検索したり、4語以上をまとめて検索して `MAX_RESULTS = 3` の上限を超えたりすると、検索漏れが起きるはずである。このことから、元の docstring で「調べたい用語を1つだけ入れる」と書かれていたのは、ツールの検索実装に合わせて検索漏れを防ぐための重要な設計だったのだと気づいた。