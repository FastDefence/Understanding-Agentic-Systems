# AI Agent Systems Lab

LLMエージェントを操り、シバき、最終的には仕事を丸ごと任せられるシステムを作るための学習・実験リポジトリです。

## Goal

最終目標は、AIを単なるコーディング支援として使うのではなく、

> **AIを用いたITシステム開発、とくに Agent Systems / Agent Infrastructure / Context Engineering の領域で、システム全体を設計・制御できるようになること**

です。

将来的には、

```text
世の中・業務のドメイン知識
        ↓
      要求
        ↓
      要件
        ↓
  システム設計
        ↓
    タスク分解
        ↓
 ┌──────┼──────┐
 ↓      ↓      ↓
Agent  Agent  Agent
 └──────┼──────┘
        ↓
      評価
        ↓
      実装
        ↓
     運用・観測
        ↓
      改善
```

という一連の工程を、複数のAgentを組み合わせたAgentic Systemとして実現することを目指します。

人間がすべてのコードを書くのではなく、

* 何を作るのか
* どこで責務を分けるのか
* 各AgentへどのContextを渡すのか
* Agent間でどの情報を受け渡すのか
* 状態をどこへ保存するのか
* 出力をどう評価するのか
* 失敗した場合にどこまで戻すのか

を設計することに重点を置きます。

## Motivation

コーディングそのものは、ITシステムを実現するための手段の一つです。

LLMによってコード生成のコストが下がるなら、人間が見るべき対象は個々のコードだけではなく、より上位のシステム全体へ移っていくと考えています。

```text
コードを書く
    ↓
コンポーネントを作る
    ↓
コンポーネントを接続する
    ↓
システム全体を設計する
    ↓
Agentを含むシステムを設計する
```

このリポジトリでは、特に以下の問題を扱います。

### Context Boundary

巨大な問題を、どこで分割するべきか。

各Agentにすべての情報を渡すのではなく、そのAgentの責務に必要なContextだけを渡す方法を考えます。

### Interface

Agent間で何を受け渡すべきか。

会話履歴を丸ごと共有するのではなく、

```text
requirements.json
architecture.json
tasks.json
progress.json
evaluation.json
```

のような構造化されたArtifactによる情報伝達を試します。

### State

AgentのContext Windowを超える長時間の仕事を、どのように継続させるかを考えます。

```text
Agent Session A
      ↓
   Artifact
      ↓
Agent Session B
      ↓
   Artifact
      ↓
Agent Session C
```

という形で、記憶ではなく外部状態によって仕事を引き継げる構造を目指します。

### Verification

Agentが「できました」と言ったことを信用するのではなく、

```text
Unit Test
Integration Test
E2E Test
Lint
Type Check
Security Check
Requirement Evaluation
```

などによって、可能な限り機械的に評価します。

## Workflow and Agent

このリポジトリでは、Anthropicの分類を基準として考えます。

### Workflow

処理経路を人間が事前に決めます。

```text
Input
 ↓
Step A
 ↓
Step B
 ↓
Step C
```

LLMは各処理を担当しても、制御フローそのものはプログラム側が持ちます。

### Agent

目的を与えられたLLM自身が、

* 次に何をするか
* どのToolを使うか
* 追加調査が必要か
* サブタスクへ分解するか
* 再試行するか

を動的に判断します。

```text
Goal
 ↓
LLM
 ├─ Tool A
 ├─ Tool B
 ├─ Tool C
 └─ Think / Retry / Delegate
```

大雑把には、

```text
Workflow = 制御フローを人間が持つ
Agent    = 制御フローをLLMが持つ
```

と捉えます。

実際のAgentic Systemでは、この2つを組み合わせます。

## Basic Patterns

まずは以下の基本構造を理解します。

### Prompt Chaining

前の処理結果を次の処理へ渡すパイプラインです。

```text
A → B → C
```

後続処理が前段の結果を必要とする場合に利用します。

### Routing

入力を分類して、適切な処理へ振り分けます。

```text
       ┌→ A
Input ─┼→ B
       └→ C
```

### Parallelization

独立した処理を並列実行します。

```text
       ┌→ A ─┐
Input ─┼→ B ─┼→ Aggregate
       └→ C ─┘
```

事前に分割方法が分かっている、比較的定型的な処理に向いています。

### Orchestrator-Workers

Orchestratorが入力を見て、必要な仕事を動的に分解します。

```text
          Orchestrator
        ┌─────┼─────┐
        ↓     ↓     ↓
     Worker Worker Worker
        └─────┼─────┘
              ↓
          Synthesis
```

複雑で、事前に必要なサブタスクを予測できない問題を扱います。

### Evaluator-Optimizer

成果物を評価し、不十分なら改善します。

```text
Generator
    ↓
Evaluator
    ↓
 PASS ─→ Done
    │
   FAIL
    ↓
Optimizer
    ↓
Generator
```

成果物の品質を継続的に改善するために利用します。

## Month 1

最初のテーマは、

> **Agentとは実装上どのようなシステムなのか**

を理解することです。

以下を自分の言葉で説明し、実装できる状態を目標とします。

```text
LLM
├─ Instructions
├─ Context
├─ Tools
├─ State / Memory
└─ Agent Loop
```

最初はAgent Frameworkによって内部処理を隠さず、LLM APIを直接呼び出して小さなWorkflowを作ります。

```text
User
 ↓
Planner
 ↓
Structured Output
 ↓
Worker
 ↓
Result
 ↓
Evaluator
 ↓
PASS / RETRY
```

重要なのは、

```text
どのAgentに
どのContextを渡し
何を出力させ
どう次へ渡すか
```

を理解することです。

## First PoC

最初の大きな成果物として、

> **「AIに全部任せてWebアプリを作ってみた」**

というPoCを作り、10〜20分程度で読める技術記事として公開します。

人間が決めるのはWebアプリのテーマです。

その後、

```text
Human
「こんなシステムが欲しい」
        ↓
Requirement Agent
        ↓
requirements
        ↓
Architecture Agent
        ↓
architecture
        ↓
Planner
        ↓
task DAG
        ↓
Orchestrator
 ┌──────┼──────┐
 ↓      ↓      ↓
Front  Back   Infra
Agent  Agent  Agent
 └──────┼──────┘
        ↓
Evaluator
        ↓
      App
```

までをAI側に実行させます。

特に検証したいのは、

> **要求から実装までの工程を、Contextを分割した複数AgentとArtifactによる情報伝達だけで成立させられるか**

です。

可能であれば、

```text
Single Agent
vs
Multi-Agent + Structured Artifacts
```

も比較します。

評価対象として、

* 要件達成率
* テスト成功率
* 要件漏れ
* 人間介入回数
* Token使用量
* APIコスト
* 実行時間
* 再試行回数

などを記録します。

## Long-term Topics

PoC以降は、以下へ進みます。

```text
Agent Fundamentals
        ↓
Context Engineering
        ↓
Long-running Agents
        ↓
Evals
        ↓
Multi-Agent Orchestration
        ↓
Agent Runtime / Harness
        ↓
Distributed Systems
        ↓
MCP / A2A
        ↓
Agent Infrastructure
```

特に、

* Context Engineering
* Artifact設計
* Agent Orchestration
* Evals
* Long-running Agents
* State Management
* Observability
* Retry / Idempotency
* Sandboxing
* MCP
* A2A
* Agent Infrastructure

を重点的に扱います。

## Philosophy

Agentを人格として捉えません。

```text
Input
 ↓
Context Assembly
 ↓
LLM
 ↓
Tool Execution
 ↓
State Update
 ↓
Evaluation
 ↓
Next Action
```

というソフトウェアコンポーネントとして扱います。

「AIに何を言えば賢く動くか」だけではなく、

> **AIを含むシステムを、どう設計すれば観測可能・制御可能・検証可能になるか**

を考えます。

また、複雑なAgentを作ること自体を目的にしません。

Workflowで十分ならWorkflowを使います。

Single Agentで十分ならMulti-Agent化しません。

Agent化によって、

* Contextを適切に分離できる
* 並列化できる
* 責務を分離できる
* 長時間処理を継続できる
* 評価可能性が上がる

などの明確な利点がある場合にのみ、構造を複雑化します。

## References

最初に以下を読みます。

### Anthropic

* Building effective agents

  * https://www.anthropic.com/engineering/building-effective-agents

* Effective context engineering for AI agents

  * https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents

* Effective harnesses for long-running agents

  * https://www.anthropic.com/engineering/effective-harnesses-for-long-running-agents

* Demystifying evals for AI agents

  * https://www.anthropic.com/engineering/demystifying-evals-for-ai-agents

* How we built our multi-agent research system

  * https://www.anthropic.com/engineering/multi-agent-research-system

### OpenAI

* Harness engineering

  * https://openai.com/index/harness-engineering/

### Other

* METR Task-Completion Time Horizons

  * https://metr.org/time-horizons/

* Model Context Protocol

  * https://modelcontextprotocol.io/

* Agent2Agent Protocol

  * https://a2a-protocol.org/

* Stanford CS336: Language Modeling from Scratch

  * https://cs336.stanford.edu/

## End Goal

最終的には、

```text
世界・業務
   ↓
要求・要件
   ↓
Agentic System
   ↓
設計
   ↓
実装
   ↓
評価
   ↓
運用
   ↓
観測
   ↓
改善
```

というループを設計し、

**「コードを書く人」ではなく、「AIが仕事を遂行できるITシステムそのものを設計できる人」**

になることを目指します。
