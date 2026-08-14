# Augmented LLM

参考: Anthropic「Building effective agents」

Anthropicでは、Agentic Systemの基本部品を次のように説明している。

```text
LLM
├─ Retrieval
├─ Tools
└─ Memory
```

単純にPromptを受け取って文章を返すLLMではなく、外部から情報を取得したり、外部システムを操作したり、過去の情報を保持・再利用したりできるLLMを **Augmented LLM** と呼んでいる。

この先のWorkflowやAgentでは、基本的に各LLM Callがこれらの能力へアクセスできるものとして考える。

---

## Augmented LLMとは

普通のLLM Callは、

```text
Input
↓
LLM
↓
Output
```

で終わる。

Augmented LLMでは、

```text
Input
↓
Context
↓
LLM
├─ Retrieval
├─ Tools
└─ Memory
↓
Output / Action
```

となる。

LLM自身にすべての能力が内蔵されているわけではない。

LLMが外部システムを直接操作するのではなく、

```text
LLM
↓
「このToolを使いたい」
↓
Runtime
↓
Tool実行
↓
外部システム
↓
Result
↓
Contextへ追加
↓
LLM
```

という構造になっている。

つまり、Agentによる「思考」をシステムとして見ると、

```text
State + External Sources
        ↓
Context Assembly
        ↓
LLM
        ↓
Action / Tool Call
        ↓
外部操作・情報取得
        ↓
State Update
        ↓
次のContext
```

という処理になる。

---

# Retrieval

Retrievalは、**外部に存在する情報から、現在の処理に必要な情報を取得する能力**である。

例えば、

```text
Web
社内Document
Vector DB
Google Drive
GitHub
Database
```

などから情報を取得する。

```text
User Query
↓
検索
↓
関連情報取得
↓
Contextへ追加
↓
LLM
```

RAGもこのRetrievalと強く関係する。

かなり単純化すると、

```text
RAG
= 外部知識を検索してContextへ入れる仕組み
```

と考えられる。

Agentになると、

```text
LLM
↓
「検索が必要」
↓
Retrieval
↓
検索結果
↓
「まだ情報が足りない」
↓
追加Retrieval
```

のように、検索するかどうか、何を検索するか、追加検索するかまで動的に判断できる。

---

# Tools

Toolは、

> **Agentから呼び出せるように公開された、意味のある操作単位**

と考える。

例えば、

```text
search_web()
read_file()
write_file()
query_database()
create_pull_request()
run_test()
send_email()
```

などである。

Toolは必ずしも外部APIそのものではない。

```text
Tool
↓
Python Function
↓
API / SDK / CLI / DB / Filesystem
↓
外部システム
```

という構造になっていることが多い。

例えば、

```text
search_google_drive()
↓
Google Drive SDK
↓
Google Drive API
↓
Google Drive
```

となる。

したがって、

> **Toolは、既存APIや処理をAgent向けの意味のある単位にラップしたもの**

と考えると分かりやすい。

---

## Tool InterfaceとTool Implementation

Toolについて混乱しやすかったのが、

```text
JSON
Python
API
```

の関係である。

ここは明確に分ける。

### Tool Interface

LLMに、

> 「こういうToolが使用できます」

と教えるための仕様。

例えば、

```json
{
  "name": "search_web",
  "description": "Web上の情報を検索する",
  "parameters": {
    "query": {
      "type": "string"
    }
  }
}
```

LLMは、

```text
Tool名
description
parameter
parameterの説明
```

をContextの一部として読み取る。

そして、

```text
「今回の処理にはWeb検索が必要」
↓
search_webを選択
↓
queryを生成
```

という判断を行う。

つまり、このJSONは、

> **LLMが自律的にToolを選ぶための説明書**

である。

---

### Tool Implementation

Toolの実体は普通のプログラムである。

例えばPythonなら、

```python
def search_web(query: str):
    return search_api.search(query)
```

となる。

Runtime側では概念的に、

```python
tool_handlers = {
    "search_web": search_web,
}
```

のようにTool名と実装を紐付ける。

LLMが、

```text
search_web(query="Anthropic agents")
```

を要求すると、

```text
LLM
↓
search_webを選択
↓
Runtime
↓
tool_handlers["search_web"]
↓
Pythonのsearch_web()
↓
検索API
```

と処理される。

重要なのは、

```text
"description": "Web上の情報を検索する"
```

をRuntimeが読んで、

```text
「Google Search APIを呼べばいいのか」
```

と判断しているわけではないことである。

`description` を解釈するのはLLMである。

その後の、

```text
search_web
↓
どのPython関数を実行するか
↓
どのAPIを呼ぶか
```

は通常のプログラムとして明示的に実装されている。

まとめると、

```text
Tool Interface
= LLM向けの説明書

Tool Implementation
= 実際に動く処理

Runtime
= InterfaceとImplementationを接続する
```

となる。

---

## Agentはどこまで選択するのか

Agentが選択できる範囲は、**どこまでをToolとして公開するか**で決まる。

例えば、

```text
search_web()
```

だけを公開した場合、

```text
Agent
↓
search_webを選択

Tool内部
↓
Google Search APIを利用
```

となる。

この場合、Googleを使うかBingを使うかはAgentは知らない。

一方、

```text
search_google()
search_github()
search_google_drive()
search_arxiv()
```

まで分けてToolとして公開すれば、

```text
Agent
↓
どの検索先を利用するか判断
```

まで自律性を広げられる。

つまり、

> **どこまでをToolとして公開するか = Agentにどこまで判断させるか**

である。

Toolの粒度はAgentの自律性、安全性、観測性に直結する。

---

# Memory

Memoryは、単純に「Contextを保存している場所」ではない。

より正確には、

> **現在のContext外に保持しておき、後の処理で再利用できる情報**

である。

例えば、

```text
過去の会話の要約
ユーザー設定
以前の調査結果
過去に得た知識
以前のタスク結果
```

などを保持できる。

必要になったら、

```text
Memory
↓
必要な情報を検索・選択
↓
Contextへ追加
↓
LLM
```

とする。

逆方向に、

```text
Context
↓
「これは後で必要」
↓
Memoryへ保存
```

という処理も作れる。

ただし、MemoryとContextが勝手に相互アクセスするわけではない。

実際にはRuntimeやContext Assembly処理が、どの情報を保存し、どの情報を現在のContextへ入れるかを管理する。

---

# Context

Contextは、

> **今回のLLM Callで、LLMが実際に見る情報の集合**

である。

例えば、

```text
Context
├─ System Instructions
├─ User Prompt
├─ Conversation History
├─ Tool Definitions
├─ Tool Results
├─ Retrieval Results
└─ Memoryから取得した情報
```

となる。

Memoryに大量の情報が存在していても、今回必要な情報だけを取り出してContextへ入れる。

```text
Memory
├─ Pythonを使う
├─ TASK-001完了
├─ 好きなEditor
├─ 過去の調査結果A
└─ 過去の調査結果B

        ↓ 必要なものだけ

Context
├─ Pythonを使う
├─ TASK-001完了
└─ TASK-002の内容
```

という感じである。

ユーザー側から見るとMemoryとContextはかなり同一に見えるが、内部では役割が違う。

```text
Memory
= 保存してある情報

Context
= 今回LLMに見せている情報
```

作業中のテーブルに近いのがContextである。

---

# State

StateはMemoryよりも広い。

```text
State
├─ Memory
├─ Runtime State
└─ External State
```

例えば、

```text
Memory
- 過去の会話
- 過去の調査結果

Runtime State
- current_task
- retry_count
- Worker Result
- PASS / FAIL

External State
- Databaseの値
- GitHub PRの状態
- Fileの内容
```

などが存在する。

例えば、

```python
retry_count = 2
current_task = "TASK-003"
```

はAgentic SystemのStateではあるが、長期保存する必要がなければMemoryには入らない。

したがって、

```text
State
= システム全体の「現在どうなっているか」

Memory
= 後から再利用するために保持する情報

Context
= 今回のLLM Callで実際に見せる情報
```

と整理できる。

---

# State・External Sources・Context・LLM

ここまでをまとめると、

```text
State
= 現在のシステム状態

External Sources
= Web / DB / Google Drive / GitHubなど

Memory
= 保持している再利用可能な情報

Context
= それらから現在必要なものだけ集めたworking set

LLM
= Contextを見て次のActionを決める

Tools
= Actionを実行する手段
```

となる。

つまり、

> **状態があって、外部ソースがあって、それによってAgentによる思考が成り立つ。**

システムとしては、

```text
State + External Sources
        ↓
必要な情報を取得
        ↓
Context
        ↓
LLM
        ↓
Tool / Action
        ↓
State変更
        ↓
Context再構築
```

というLoopになる。

---

# Tailoring

AnthropicはAugmented LLMを実装するときに、

> tailoring these capabilities to your specific use case

を重視している。

これは、

```text
どんなRetrievalを与えるか
どんなToolsを与えるか
どんなMemoryを持たせるか
Toolをどの粒度で公開するか
どんな情報をContextへ入れるか
```

を用途に合わせて設計するという意味である。

例えばGitHub関連のAgentなら、

```text
search_web()
```

のような汎用Toolだけではなく、

```text
get_issue()
get_pull_request()
get_diff()
post_review_comment()
```

のように、用途に合わせたTool Interfaceを用意できる。

LLMにとって、

```text
何のToolなのか
いつ使うのか
どんなArgumentが必要なのか
```

が分かりやすいInterfaceを設計することが重要になる。

---

# MCP

これまでの構造では、外部サービスごとに、

```text
Agent
↓
Tool Interface
↓
Tool Implementation
↓
各サービスのAPI
```

という接続を作る必要がある。

MCPは、このAgentと外部Tool・Data Sourceを接続する部分を共通化するための仕組みとして捉えられる。

```text
Agent / LLM
↓
MCP Client
↓
MCP Server
↓
Google Drive / GitHub / DB / etc.
```

つまり、

> **Toolや外部Data Sourceとの接続方法を標準化する**

方向の技術である。

Month 1ではMCP自体には深く入らず、

```text
今まで個別実装していたTool接続を
共通Interfaceで扱うための仕組み
```

程度の理解に留める。

---

# ここまでの理解

Agentic Systemの基本部品は、

```text
Augmented LLM
├─ Retrieval
├─ Tools
└─ Memory
```

である。

しかし実装レベルまで見ると、

```text
State
+
External Sources
↓
Context Assembly
↓
LLM
↓
Tool Selection
↓
Tool Runtime
↓
API / SDK / DB / Filesystem
↓
State Update
```

という普通のシステムとして構成されている。

特に重要なのは、

```text
Tool Schema
= LLMがToolを選択するためのInterface

Tool Implementation
= 実際の処理

Context
= 現在LLMが見ている情報

Memory
= 後から再利用するために保持する情報

State
= システム全体の現在状態
```

という区別である。

Agentは人格として存在しているわけではない。

> **現在状態と外部情報からContextを作り、LLMが次のActionを決定し、Toolによる実行結果でStateを更新するシステム**

として考える。
