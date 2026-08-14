# Agentic Systems の基本

参考: Anthropic「Building effective agents」

この文献では、複雑なFrameworkの使い方から入るのではなく、LLMを含むシステムをどのように構造化するかという基本原理から説明している。

なお、記事冒頭には2024年12月以降にTooling環境が大きく変化したという注記がある。そのため、個別のSDKやToolingについては最新資料を参照しつつ、この文献ではAgentic Systemの設計原理を中心に読む。

## What are agents?

Anthropicでは、LLMを用いたこれらのシステムを広く **Agentic Systems** と呼び、その中を大きく `Workflow` と `Agent` に分けている。

```text
Agentic Systems
├─ Workflow
└─ Agent
```

### Workflow

Workflowは、LLMやToolを使う処理経路が、あらかじめコード側で定義されているシステムである。

例えば、

```text
Input
↓
LLM A
↓
LLM B
↓
Output
```

という処理順序をPythonなどで固定している場合、複数のLLMを使用していたとしてもWorkflowである。

### Agent

Agentでは、LLM自身が状況を見ながら、

```text
何をするか
どのToolを使うか
次に何をするか
いつ終了するか
```

などを動的に判断する。

例えば、

```text
Input
↓
LLM
↓
「検索が必要」
↓
Search Tool
↓
検索結果
↓
LLM
↓
「もう一度調べる必要がある」
↓
Search Tool
↓
LLM
↓
Output
```

のような処理になる。

Agentだからといって複数のAgentが必要なわけではない。

```text
複数LLMが存在する
≠ Agent
```

複数のLLMが存在しても処理経路が固定されていればWorkflowであり、逆にLLMが1つだけでも、自分でToolや次の処理を選択していればAgentになり得る。

したがって、WorkflowとAgentを区別するときには、

> **次の処理を誰が決めているか。コードか、LLMか。**

を見ると分かりやすい。

---

## When and when not to use agents

Anthropicは、まず可能な限り単純な方法を使い、必要になってから複雑性を増やすことを推奨している。

つまり、

```text
単純なLLM Call
↓ 必要なら
Workflow
↓ さらに必要なら
Agent
```

という順番で考える。

Agentic Systemを使えば常に良いというわけではない。

Agentは、

```text
考える
↓
Toolを使う
↓
結果を見る
↓
また考える
↓
必要なら繰り返す
```

という処理になりやすいため、単純なLLM Callや固定Workflowと比較して、

* LLM Call数が増える
* Token消費量が増える
* Latencyが増える
* API Costが増える
* 実行量を予測しづらくなる

という代償がある。

言い換えると、

> **Workflowのほうが電気代かかんなそう。**

という感覚はかなり正しい。

その代わりAgentは、事前にすべての処理経路を決められない問題へ柔軟に対応できる。

したがって、

```text
予測可能な部分
→ Workflow

不確定な部分
→ Agent
```

と分けるのが合理的である。

より端的には、

> **決め打ちできる場所はWorkflow、判断が必要な場所だけAgent。**

となる。

全部をAgent化するのではなく、必要な場所にだけLLMの自律性を置く。

これはAgent特有の話というより、責務分離や構造化と同じシステム設計上の考え方である。

---

## When and how to use frameworks

Agentic Systemを構築するためのFrameworkは多数存在する。

Frameworkを使用すると、

```text
LLM API Call
Tool定義
Tool CallのParse
Workflow接続
State管理
Retry
```

などの処理を簡単に実装できる。

一方で、Frameworkによる抽象化が増えるほど、

```text
実際にどんなPromptが送られたのか
どんなContextが渡されたのか
LLMが何を返したのか
なぜそのToolが選択されたのか
どこで失敗したのか
```

が見えづらくなる。

また、Frameworkが便利だからという理由だけで、本来必要のない複雑なAgent構成を作ってしまう可能性もある。

そのため、最初はLLM APIを直接呼び出す形で実装する。

例えば、

```python
planner_output = call_llm(...)
worker_output = call_llm(...)
evaluation = call_llm(...)
```

くらい露骨に書く。

Agentic Systemは特殊な魔法ではなく、かなり単純化すると、

```text
LLM API
+
普通のプログラムの制御構造
+
Context
+
State
+
Tools
```

の組み合わせである。

Cookbookに存在する `chain()`、`route()`、`parallel()` なども、基本的にはLLM API CallをPythonの制御構造で組み合わせたものになっている。

したがってMonth 1では、Frameworkの使い方を覚えることよりも、

```text
どのLLMに
何をContextとして渡し
何を出力させ
次の処理へどう接続するか
```

を理解することを優先する。

Frameworkはその構造を理解した後で使えばよい。

---

## ここまでの理解

この時点では、Agentic Systemを次のように捉える。

```text
Agentic System
= LLMをシステムの構成要素として利用する仕組み

Workflow
= 処理経路をコードが決める

Agent
= 処理経路やTool利用をLLMが動的に決める
```

重要なのは、

> **LLMをどう使うかではなく、LLMを含むシステムをどう制御するか。**

という視点である。

この先では、Agentic Systemの基本部品である `Augmented LLM` と、その上に構築される各種Workflowについて見る。
