# 第3回 解答

## 演習

- 第1回（素のLLM）とエージェントの回答の違い：
用語集を検索し、その内容を基に回答するようになった。

＜実行結果＞
@Shichiken ➜ /workspaces/lecture-ai-engineering-2026 (lec03) $ uv run python -m app.agent "AIE2026の講座で使う「lecNN-start タグ」とは何ですか？"
[model] gemini-3.5-flash-lite
「lecNN-start タグ」とは、AIE2026の講座リポジトリにおいて、各回の演習開始時点のコードに付けられているGitタグのことです（例: `lec03-start`）。

前の回の内容が終わっていなくても、このタグから新しくブランチを作成することで、その回の演習をスムーズに始められるようになっています。

関係する回：第1回
@Shichiken ➜ /workspaces/lecture-ai-engineering-2026 (lec03) $ uv run python -m app.agent --trace "LoRAとQLoRAの違いは？"
[model] gemini-3.5-flash-lite
LoRAとQLoRAの違いは、ベースモデルの重みをそのまま使うか、量子化してメモリを節約するかにあります。

- **LoRA**
  事前学習済みの重みは固定したまま、重みの更新分を2つの小さな行列（低ランク行列）の積で表し、その小さな行列だけを学習するPEFT手法です。学習パラメータや保存ファイルを削減できます。
- **QLoRA**
  ベースモデルの重みを4ビットなどに**量子化**してメモリを大幅に節約した状態で固定し、その上でLoRAと同様の仕組みで学習を行う手法です。これにより、より大きなモデルを比較的小さなGPUメモリで扱えるようになります。

関係する回：第2回

--- trace ---
[tool call]   search_glossary({'query': 'LoRA'})
[tool call]   search_glossary({'query': 'QLoRA'})
[tool return] search_glossary -> [{"term": "LoRA", "lecture": 2, "definition": "事前学習済みの重みは固定したまま、重みの更新分を2つの小さな行列の積（低ランク行列）で表し、その小さな行列だけを学習するPEFT手法。学習するパラ…
[tool return] search_glossary -> [{"term": "QLoRA", "lecture": 2, "definition": "ベースモデルの重みを4ビットなどに量子化してメモリを節約したうえで、LoRAで学習する手法。より大きなモデルを小さなGPUで扱える。本講座の第2…
[usage] RunUsage(cost=Decimal('0.0008513'), details={'text_prompt_tokens': 1196}, input_text_tokens=1196, input_tokens=1196, output_tokens=197, requests=2, tool_calls=2)
@Shichiken ➜ /workspaces/lecture-ai-engineering-2026 (lec03) $ uv run python -m app.agent --list-tools
モデルに見えているツール: 3 個

[search_glossary]
  講座の用語集から用語を検索し、定義と、どの回で扱うかを返す。
  
  AIエンジニアリングの用語や、この講座（AIE2026）に固有の言葉について
  答える前に使うこと。query には調べたい用語を1つだけ入れる（例: "LoRA"）。
  引数: {"query": {"type": "string"}}

[get_lecture_summary]
  指定した回（1始まりの整数）の講義の要点を返す。
  
  「第3回では何を学ぶ？」のように、回の内容や関係を聞かれたときに使う。
  引数: {"lecture": {"type": "integer"}}

[make_quiz]
  指定した回の用語から1つを選び、確認問題と答えを返す。
  
  「第2回の確認問題を出して」のように、復習用の問題を求められたときに使う。
  引数: {"lecture": {"type": "integer"}}


- `--trace` で見えたツール呼び出し（ツール名と引数）：
[tool call]   search_glossary({'query': 'LoRA'})
[tool call]   search_glossary({'query': 'QLoRA'})


- 1つの質問での LLM 呼び出し回数（`requests`）：2
[usage] RunUsage(cost=Decimal('0.0008513'), details={'text_prompt_tokens': 1196}, input_text_tokens=1196, input_tokens=1196, output_tokens=197, requests=2, tool_calls=2)


- 追加したツールと、docstring に書いた説明：
def make_quiz(lecture: int) -> str:
    """指定した回の用語から1つを選び、確認問題と答えを返す。

    「第2回の確認問題を出して」のように、復習用の問題を求められたときに使う。
    """
    terms = [entry for entry in load_glossary() if entry["lecture"] == lecture]
    if not terms:
        return f"第{lecture}回の用語は用語集にありません。"
    term = random.choice(terms)
    return json.dumps(
        {"question": f'{term["term"]}の意味は？', "answer": term["definition"]},
        ensure_ascii=False,
    )



## 宿題

- 取り組んだレベル：実装
- やったこと：
(1)tools.pyに以下の学習記録をログファイルに出力する関数を作成

def save_learning_log(content: str) -> str:
    """利用者が依頼した学習の記録を learning_log/ に保存する。

    学んだことや振り返りを記録として残すよう頼まれたときに使う。
    保存前に利用者の承認が必要。
    """
    LEARNING_LOG_DIR.mkdir(parents=True, exist_ok=True)
    filename = datetime.now().strftime("%Y%m%d-%H%M%S-%f") + ".md"
    path = LEARNING_LOG_DIR / filename
    with path.open("x", encoding="utf-8") as f:
        f.write(content)
    return f"学習の記録を保存しました: {filename}"

(2)app/agent.pyを以下のように修正

- 承認待ちを扱えるよう変更：`Agent` の出力型に `DeferredToolRequests` を追加しました。承認が必要なツールを呼んだとき、すぐ実行せず保留状態を返せるようにするためです。
- ツールを登録：`make_quiz` に加え、`save_learning_log` を `requires_approval=True` で登録しました。
- 実行時に承認を確認：保留中の呼び出しがあれば、CLIで `[y/N]` と確認します。`y` または `yes` の場合のみ承認し、会話履歴と承認結果を渡してエージェントを再実行します。それ以外は拒否になります。

- 結果：
(1)通常の質問をした場合（ログ出力を依頼しなかった場合）は、通常の回答を返却した

@Shichiken ➜ /workspaces/lecture-ai-engineering-2026 (lec03) $ uv run python -m app.agent "AIE2026の講座の宿題にある「3レベル」とは何ですか？"
[model] gemini-3.5-flash-lite
AIE2026の講座における「3レベル」とは、宿題の構成を示す3つの段階のことです。

具体的には以下のようになっています：

1. **基礎**：講義内容の確認を行うレベル
2. **実装**：自分でコードを書くレベル
3. **発展**：調査や試行を行うレベル

3つのうち**どれか1つ**に取り組めば、その回の宿題は完了となります。なお、原則として採点は行われません。

（第1回）

(2)ログファイルを明示して依頼した場合は、学習記録を保存するか確認メッセージが表示され、yを選択するとログファイルが出力された。

@Shichiken ➜ /workspaces/lecture-ai-engineering-2026 (lec03) $ uv run python -m app.agent "AIE2026の講座の宿題にある「3レベル」とは何ですか？結果をログファイル に出力して"
[model] gemini-3.5-flash-lite
save_learning_log を実行して学習の記録を保存しますか？ [y/N]: y
AIE2026の講座における「宿題の3レベル」とは、宿題の構成を**「基礎」「実装」「発展」**の3段階に分けたものです。

* **基礎**：講義内容の確認
* **実装**：自分でコードを書く
* **発展**：調査や試行

これら3つのうち**どれか1つ**に取り組めば、その回の宿題は完了となります。原則として採点は行われません。

（第1回）

(3)ログファイルの結果は以下の通り
AIE2026の宿題の「3レベル」についての質問と回答。3レベルは「基礎」「実装」「発展」の3段階で構成され、いずれか1つに取り組めば完了となる。


- 気づき・疑問：
エージェントに関数の利用方法を正確に指示するためには、関数のdocstringに「いつ何のために使うのか、何を実施してほしいのか」など５Ｗ１Ｈを具体的に記載することが重要であることが理解できた。

