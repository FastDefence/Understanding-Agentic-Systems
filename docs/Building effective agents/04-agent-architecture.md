# Agents

参考: Anthropic「Building effective agents」

ここまで、

```text
Prompt Chaining
Routing
Parallelization
Orchestrator-Workers
Evaluator-Optimizer
```

という5つのWorkflowを見てきた。

これらは、LLMを含む処理を構造化するための代表的なPatternである。

```text
Prompt Chaining
= 工程化・順次処理

Routing
= 職能別分掌・分岐

Parallelization
= 定型的な並列分業

Orchestrator-Workers
= 動的な分業設計

Evaluator-Optimizer
= レビューと改善ループ
```

一方、Agentでは、事前に処理経路を固定できない部分について、LLM自身がEnvironmentから得られる情報を見ながら次のActionを決定する。

Anthropicは、LLMの複雑な入力理解、reasoning / planning、Tool利用、ErrorからのRecovery能力が向上したことで、AgentがProductionでも使われ始めているとしている。

---

## Agentとは

Agentはかなり単純化すると、

> **EnvironmentからFeedbackを受けながら、LLMがToolを使って次のActionを自分で決めるLoop**

である。

```text
Human
↓
Task / Command
↓
Agent
↓
Plan / Actionを決定
↓
Tool
↓
Environment
↓
Ground Truth
↓
Agent
↓
次のAction
↓
...
↓
Finish
```

Workflowとの大きな違いは、

```text
Workflow
= 次の処理をコードが決める

Agent
= 次の処理をLLMが動的に決める
```

ことである。

例えば、

```text
Search
↓
Analyze
↓
Write
```

という処理順序が最初から固定されているならWorkflowである。

一方、

```text
調査する
↓
「情報が足りない」
↓
追加検索
↓
「別の情報源も必要」
↓
別Toolを使う
↓
分析
↓
「十分な情報が集まった」
↓
回答
```

と、実行中の結果によって次の処理そのものが変化するならAgentである。

AnthropicもAgentを、EnvironmentからのFeedbackに基づいてToolをLoopの中で利用するLLMとして説明している。

---

## なぜ5つのWorkflow Patternの後にAgentが出てくるのか

この記事では、いきなりAgentを説明していない。

まず、

```text
Prompt Chaining
Routing
Parallelization
Orchestrator-Workers
Evaluator-Optimizer
```

という構造化可能なWorkflowを説明した後にAgentが登場する。

この順番は、

```text
まず構造化できる部分はWorkflowとして構造化する
↓
それでも実行経路を事前に固定できない部分がある
↓
そこでAgentを使う
```

と考えると分かりやすい。

例えば、

```text
「このコードを
セキュリティ・性能・可読性からレビューして」
```

であれば、

```text
Security
Performance
Readability
```

という分割を事前に決められるため、Parallelizationで処理できる。

一方、

```text
「このGitHub Issueを解決して」
```

というTaskでは、

```text
どのFileを見るか
何File変更するか
どのTestを実行するか
失敗したらどこを見るか
追加調査が必要か
何回修正するか
```

を事前にHard Codeすることが難しい。

そこで、

```text
Action
↓
Environment
↓
Ground Truth
↓
次のAction
```

というAgent Loopが必要になる。

したがって、

> **不確実性があるからAgentを使う**

だけでは少し足りない。

より正確には、

> **必要なStep数や処理経路を事前にHard Codeできず、実行中に得られるEnvironmentのFeedbackを見ながら次のActionを決める必要がある場合にAgentを使う。**

Anthropicも、必要なStep数を予測しづらく、固定経路をHard CodeできないOpen-ended ProblemをAgentの適用対象としている。

---

# Agent Loop

AgentはHumanからCommandを受けるか、HumanとのInteractive DiscussionによってTaskを明確化してから仕事を始める。

Taskが明確になれば、Agentは基本的に自律してPlanとExecutionを進める。必要になった場合だけ再びHumanへ情報や判断を求める。

```text
Human
↓
要求を明確化
↓
Agent
↓
Plan
↓
Action
↓
Tool
↓
Environment
↓
Ground Truth
↓
進捗を判断
↓
Next Action
↓
...
```

このLoopをTaskが完了するまで繰り返す。

かなりPDCAに近く見ることもできる。

```text
Plan
→ 次のActionを決める

Do
→ Toolを実行する

Check
→ Environmentの実際の結果を確認する

Act
→ 結果を受けて修正・再計画する
```

---

# Ground Truth

Agent Loopにおいて重要なのが、Environmentから得られる **Ground Truth** である。

Anthropicは、Agentが各Stepで自分の進捗を評価するために、Tool Call ResultやCode ExecutionなどからGround Truthを取得することを重要視している。

Coding Agentなら、

```text
Agent
「これで直ったはず」
↓
pytest
↓
3 tests failed
```

というとき、

```text
3 tests failed
```

がGround Truthである。

つまり、

```text
LLMによる推測
≠ Ground Truth

実際のTool Result
Code Execution Result
Environmentの実際の状態
= Ground Truth
```

である。

Agentは、

```text
修正
↓
Test
↓
FAIL
↓
原因を考える
↓
再修正
↓
Test
```

というTrial and Errorを自律的に行える。

PDCAで考えるなら、Ground Truthは **Cそのものというより、Cで使う現実側の材料** である。

また、

```text
生のTest Log
= Ground Truth

「原因はDB接続失敗だ」
= Agentによる解釈
```

である。

Agentが生成した解釈までGround Truth扱いすると、Agent自身の誤認を事実として次の処理へ渡してしまう可能性がある。

---

# Human in the Loop

Agentは自律的に動くが、すべてを自力で判断し続ける必要はない。

Anthropicは、CheckpointやBlockerに遭遇した場合、AgentがHuman Feedbackを求めてPauseできるとしている。

例えば、

```text
Agent
↓
仕様書を読む
↓
「この仕様ではAとBのどちらか決められない」
↓
ask_human
↓
「AとBのどちらですか？」
↓
Human
↓
追加Context
↓
Agent再開
```

となる。

つまり、

> **「このContextでは情報が足りません！停止！」**

もAgentにとって重要なActionである。

人間が仕様書を読んで、

> 「この仕様書だと、ここどうすればいいか分からないです」

と言うのと近い。

より良いAgentなら、単に止まるのではなく、

```text
何が不足しているか
なぜ現在のContextでは決定できないか
どの判断をHumanにしてほしいか
```

まで明示する。

例えば、

```json
{
  "status": "NEEDS_CLARIFICATION",
  "missing_information": [
    "ログイン失敗時のHTTP Status",
    "Account Lock条件"
  ],
  "question": "401を返し、5回失敗でLockする仕様でよいですか？"
}
```

のようにできる。

これは単なる会話上の質問ではなく、

> **Agentが自律性の限界を検知し、意思決定をHumanへEscalationする制御**

と考えられる。

---

## ask_human / stop / finish

これらは分けて考える。

```text
ask_human
= 一時停止
→ 必要な情報をHumanから受け取ったら再開

stop / abort
= 処理を中止

finish
= Taskを正常完了
```

Human in the Loopは、AgentのTrial and Errorを毎回Humanが操作するという意味ではない。

基本は、

```text
Agent
↕
Environment
```

でLoopを回し、自力で解決できない地点だけHumanへ戻る。

---

# Stopping Condition

Agentは自律的にLoopできるため、

```text
Plan
↓
Action
↓
Feedback
↓
Retry
↓
Feedback
↓
Retry
↓
...
```

を長時間続ける可能性がある。

Anthropicも、Task完了以外にMaximum IterationsなどのStopping Conditionを設定することを挙げている。

例えば、

```text
Task完了
→ finish

情報不足 / Blocker
→ ask_human

MAX_ITERATIONS到達
→ stop
```

とできる。

つまり、

> **無限のPDCAを回すことはできるが、本当に無限に回してはいけない。**

自律性をAgentへ渡しても、その外側には人間が制御可能な境界を残す。

---

# Agentの実装自体は意外と単純

Agentは高度なTaskを実行できるが、その基本実装は意外と単純である。

Anthropicは、

> LLMがEnvironment Feedbackを受けながらToolをLoopで使う

という構造として説明している。

これまで整理した要素まで含めると、

```text
Context
↓
LLM
↓
Action / Tool Call
↓
Runtime
↓
Tool
↓
Environment
↓
Ground Truth
↓
State Update
↓
Context Assembly
↓
LLM
```

となる。

LLMそのものは、

```text
Context
↓
Tokenizer
↓
Transformer
↓
Token生成
```

を行うModelである。

Agentは、そのLLMを判断器として利用するSoftware Systemである。

```text
Agent
├─ LLM
├─ Instructions
├─ Context
├─ State / Memory
├─ Tools
├─ Runtime
├─ Environment
└─ Agent Loop
```

---

# ToolとAgent

AgentはEnvironmentへ直接作用しているわけではない。

基本的には、

```text
LLM
↓
Tool Call
↓
Runtime
↓
Tool Implementation
↓
API / SDK / CLI / DB / Filesystem
↓
Environment
```

となる。

LLMが、

```text
search_web(query="...")
```

のようなTool Callを出力し、Runtimeが対応するTool Implementationを実行する。

そのためAgentが利用できる能力は、

```text
どんなToolを公開するか
Toolをどの粒度にするか
Tool Interfaceをどう説明するか
```

によって大きく変わる。

Anthropicも、AgentではToolsetとそのDocumentationを明確かつ慎重に設計することが重要だとしている。

---

# When to use Agents

Agentが向いているのは、

```text
Open-ended Problem

必要なStep数が事前に分からない

実行経路をHard Codeできない

途中結果によって次のActionが変化する

長いTurnを通して自律実行する価値がある
```

ような問題である。

逆に、

```text
処理順序が固定できる
分岐条件が明確
処理を事前に分割できる
```

なら、Workflowの方が予測可能で安い。

したがって、

```text
固定できる
→ Workflow

固定できないが、
Environmentを見ながら進められる
→ Agent
```

と考える。

AgentではLLMが多Turnにわたって判断するため、その意思決定をある程度TrustできるEnvironmentで使う必要がある。

---

# CostとCompounding Errors

Agentは自律的にLoopするため、

```text
LLM Call数
Token消費
Tool Call数
実行時間
```

が増えやすい。

そのためWorkflowよりCostが高くなりやすい。

さらにAgentでは、

```text
小さな誤判断
↓
誤ったAction
↓
Environment / Stateが変化
↓
その結果を基に次の判断
↓
さらにズレる
```

という **Compounding Errors** が起こり得る。

そのためAnthropicは、

```text
Sandbox
Guardrails
Extensive Testing
```

を推奨している。

つまり、

```text
Agentに自律性を与える
≠
無制限な権限を与える
```

である。

---

# Agentが有効な例

## Coding Agent

Anthropicは、SWE-benchのTaskを解決するCoding Agentを例として挙げている。

例えば、

```text
「このIssueを解決して」
```

というTaskに対して、

```text
どのFileを見るか
何File修正するか
どのTestを実行するか
失敗したら何を調べるか
追加修正が必要か
```

を事前には決められない。

そこで、

```text
Issueを読む
↓
Repositoryを調べる
↓
関係Fileを読む
↓
Code変更
↓
Test
↓
Ground Truth
↓
修正
↓
再Test
```

をAgent自身が回す。

Codingは、

```text
Codeを書く
↓
Testする
↓
明確な結果が返る
```

というGround Truthを取得しやすいため、Agentと相性がよい。

---

## Computer Use

もう1つの例が、ClaudeがComputerを操作してTaskを実行するComputer Useである。

```text
Agent
↓
画面を見る
↓
Click / Keyboard Input
↓
Computer
↓
画面が変化
↓
新しいEnvironment
↓
Agent
↓
次のAction
```

となる。

GUI操作では、

```text
現在どの画面か
次にどこをClickするか
何を入力するか
操作が成功したか
Taskが完了したか
```

が実行してみるまで分からない。

そのため固定WorkflowよりAgent Loopが向いている。

---

# Combining and customizing these patterns

Anthropicは、これまで紹介したBuilding BlockやWorkflowをPrescriptiveなものとはしていない。

Use Caseに合わせて形を変え、組み合わせて使うCommon Patternとして扱っている。

また、複雑性を追加するのは、それによってOutcomeが実際に改善するとMeasurementできた場合だけにすべきだとしている。

例えば、

```text
Routing
↓
Orchestrator
↓
Parallel Workers
↓
Evaluator
↓
必要ならAgentが追加調査
```

のように組み合わせられる。

つまり、

> **5類型は構造化定理のような代表的なモデルであって、唯一のArchitectureではない。**

---

# 1つのTask内で複数の構造は混在する

1つのUser Requestが、必ず1つのWorkflowや1つのAgentだけで処理できるとは限らない。

むしろ複雑なTaskでは、

> **構造化可能な目的と、構造化不可能な目的が1つの質問内に混ざることはザラにある。**

例えば、

```text
「このサービスの課題を調査して、
改善案を考えて、
技術的な実現可能性を確認して、
提案資料にして」
```

というTaskでは、

```text
構造化しやすい部分
- 調査 → 改善案 → 資料化
- 技術・Cost・運用などの観点別評価

構造化しにくい部分
- 何を追加調査すべきか
- どの課題が本質的か
- どの実装案を採用すべきか
- 情報不足ならどこへ戻るか
```

が同時に存在する。

したがって、

```text
1つの質問
≠ 1つのWorkflow
≠ 1つのAgent
```

である。

Taskを分解した結果、

```text
固定された工程
→ Prompt Chaining

入力によって処理先が変わる
→ Routing

独立して処理できる
→ Parallelization

Subtaskの分割方法自体が分からない
→ Orchestrator-Workers

品質を評価して改善する
→ Evaluator-Optimizer

Environmentを見ないと次のActionを決められない
→ Agent Loop
```

のように、複数の構造が1つのAgentic System内に混在し得る。

---

# 構造の動的割り当てもAgentに任せられる

ここからは、5類型とAgentの理解から得た考察であり、Anthropicの記事がそのままこのArchitectureを提示しているわけではない。

1つのTaskを処理するとき、

```text
ここはChaining
ここはParallelization
ここはAgent Loop
```

と人間が毎回設計することもできる。

一方で、この**構造の割り当て自体をAgentへ委譲する**ことも考えられる。

```text
User Request
↓
Agent / Orchestrator
↓
Taskを分解
↓
各Taskに必要な構造を判断
├─ Chaining
├─ Routing
├─ Parallelization
├─ Orchestrator-Workers
├─ Evaluator-Optimizer
└─ Agent Loop
↓
実行
↓
Ground Truth / Evaluation
↓
必要なら構造を再構成
```

つまり、

> **課題解決に必要なAgentic Systemの内部Architecture自体を、Agentがある程度動的に構成する**

こともできる。

Orchestrator-Workersはすでに、その考え方の一部を持っている。

Orchestratorは入力を見て、

```text
今回は何個のSubtaskが必要か
各Workerに何をさせるか
```

を動的に決める。これはParallelizationより一段自律性が高い。

これをさらに広げれば、

```text
今回は並列化する
ここは固定Chainingでよい
このOutputはEvaluatorへ送る
追加調査だけAgent Loopに任せる
```

という判断までAgentに委譲できる。

---

# ただし、すべてをAgentに設計させる必要はない

Agentに構造を選択させられるからといって、毎回すべてを動的にする必要はない。

明らかに固定できる処理までLLMに毎回判断させると、

```text
Cost増加
Latency増加
挙動の不安定化
Observability低下
```

につながる。

そのため、

```text
固定できる構造
→ Runtime / Workflowとして人間が用意

入力によって変化する構造
→ Agentが選択・構成

予測不可能な実行経路
→ Agent Loop

結果
→ Ground Truth / Evaluatorで確認
```

という分担が合理的である。

人間はさらに外側の、

```text
目的
成功条件
利用可能なTool
権限
予算
Stopping Condition
Guardrail
```

を握る。

一方でAgentには、

```text
Task分解
Workflowの選択
Worker構成
並列化するか
追加調査するか
Retry地点
```

などを任せられる。

したがって、

> **外側の目的・制約は人間が設計し、内側の課題解決Architectureは必要に応じてAgentが動的に構成する。**

という設計が考えられる。

---

# Workflow構造も目的ではなく手段

重要なのは、

```text
Prompt Chainingを使う
Parallelizationを使う
Agentを使う
```

こと自体を目的にしないことである。

```text
構造
= 目的
```

ではなく、

```text
構造
= 課題解決のための可変な手段
```

である。

そのため、

> **固定できるところは構造化し、固定できないところだけ自律化する。必要なら、その構造の選択自体もAgentへ委譲する。**

という考え方になる。

---

# Summary

Anthropicは、LLM Systemで成功するために、最もSophisticatedなSystemを作る必要はないとしている。

まずSimple Promptから始め、Evaluationによって改善し、それでも不足すると確認できた場合にだけMulti-stepなAgentic Systemへ進む。

```text
Simple Prompt
↓
Evaluationしながら最適化
↓
必要ならWorkflow
↓
それでも不足するならAgent
```

Agentを実装するときには、3つのPrincipleを挙げている。

```text
1. Simplicity
   Agent設計を単純に保つ

2. Transparency
   AgentのPlanning Stepを明示する

3. ACI
   Agent-Computer Interfaceを丁寧に設計・Testする
```

Frameworkは素早く開発を始めるためには有効だが、Productionへ進む過程で必要ならAbstraction Layerを減らし、基本Componentから構築してもよいとしている。

つまり、

> **Agentを作ること自体が目的ではない。必要な問題に対して、最も単純で制御可能なSystemを選ぶ。**

ということになる。

---

# Appendix 1: Agents in practice

AnthropicはAgentの有望な実用分野として、

```text
Customer Support
Coding
```

を挙げている。

この2つには、

```text
ConversationとActionの両方が必要

Success Criteriaが明確

Feedback Loopを作れる

Human Oversightを組み込める
```

という共通点がある。

---

## Customer Support

Customer Supportでは、

```text
HumanとのConversation
+
外部SystemへのAction
```

が自然に組み合わさる。

Toolによって、

```text
Customer Data取得
Order History取得
Knowledge Base検索
Refund
Ticket Update
```

などを実行できる。

そして、

```text
Problemが解決したか
```

というSuccess Criteriaを比較的明確に定義できるため、Open-endedなAgentと相性がよい。

---

## Coding Agents

CodingもAgentと相性がよい。

```text
Codeを書く
↓
Automated Test
↓
Ground Truth
↓
修正
↓
再Test
```

というFeedback Loopを作りやすい。

また、

```text
Problem Spaceが比較的構造化されている

Automated TestでFunctionalityを検証できる

Output Qualityを比較的客観的に評価できる
```

という特徴がある。

ただし、Testが通るだけでSystem全体のRequirementを満たしているとは限らない。

Anthropicも、Automated Testingに加えて、より広いSystem Requirementとの整合性を確認するHuman Reviewが依然として重要だとしている。

---

# Appendix 2: Prompt Engineering your Tools

ToolはAgentとEnvironmentのInterfaceである。

Anthropic APIではToolのStructureとDefinitionを指定し、ClaudeがToolを利用すると判断した場合、API ResponseにTool Useを返す。

そのためTool Definitionそのものにも、

> **Promptと同程度にPrompt Engineeringが必要**

である。

---

## LLMが扱いやすいTool Interfaceにする

同じActionでも、LLMにとって扱いやすい表現と扱いにくい表現がある。

例えばFile編集なら、

```text
Diffを書く
```

ことも、

```text
File全体を書き直す
```

こともできる。

Software Engineering的には相互変換可能でも、LLMにとって難易度は同じではない。

Diffでは、新しいCodeを書くより前に変更行数などを正確に扱う必要がある。

CodeをJSON Stringとして出力させる場合も、

```text
newline
quote
escape
```

などのFormatting Overheadが発生する。

AnthropicはTool Formatについて、

```text
Modelが行き詰まる前に考えられるToken余地を与える

Internet上の自然なTextに近いFormatを使う

行数の厳密なCountや大量のEscapeなど、
不要なFormatting Overheadを避ける
```

ことを推奨している。

---

## Agent-Computer Interface

Anthropicは、人間向けの **Human-Computer Interface（HCI）** と同じように、Agent向けの **Agent-Computer Interface（ACI）** にも設計労力をかけるべきだとしている。

Tool Definitionでは、

```text
Tool名
Description
Parameter名
Parameter Description
Example Usage
Edge Cases
Input Format
他Toolとの境界
```

などを、LLMが理解しやすいように設計する。

特に、

> **Teamに入ったJunior Developerへ書く良いdocstring**

くらい明確なParameter名とDescriptionを意識する。

似たToolが多数存在する場合には、どのToolをいつ使うかの境界を明確にする必要がある。

---

## ToolもTestして改善する

Toolは一度定義して終わりではない。

```text
AgentにToolを使わせる
↓
どこで間違えるか観測する
↓
Tool Interfaceを変更
↓
再Test
```

というIterationが必要になる。

Anthropicは、Toolを **Poka-yoke**、つまりAgentが間違えにくいInterfaceへ変更することも推奨している。

SWE-bench Agentでは、Relative PathをArgumentにすると、AgentがRoot Directoryから移動した後にPathを間違えることがあった。

そこでAbsolute Pathを必須にしたところ、Toolを安定して使用できるようになったという例が紹介されている。

つまり、

```text
LLMが間違えた
↓
Promptを頑張って直す
```

だけではない。

```text
LLMが間違えた
↓
Tool Interfaceそのものを間違えにくくする
```

というSoftware Engineering的な改善ができる。

---

# この記事で学んだもの

この記事では、Agentという単一の技術だけではなく、

> **Agentic Systemの登場人物と、それらをどう組むかというArchitecture**

を学んだ。

登場人物・部品としては、

```text
Human

LLM

Context

State / Memory

Retrieval

Tools

Runtime

Environment

Ground Truth

Worker

Orchestrator

Evaluator
```

などがある。

そしてArchitectureとして、

```text
Prompt Chaining
Routing
Parallelization
Orchestrator-Workers
Evaluator-Optimizer
Agent Loop
```

を学んだ。

これらを組み合わせると、

```text
Input
↓
Context Assembly
↓
LLM
↓
Action
↓
Tool Execution
↓
Environment
↓
Ground Truth
↓
State Update
↓
Evaluation
↓
Next Action
```

というAgentic Systemとして見ることができる。

重要なのは、

```text
固定できる部分
→ Workflowとして構造化する

事前に固定できない部分
→ Agentへ自律性を与える

Agentが行動した結果
→ Ground Truthで確認する

Agentだけでは判断できない
→ Humanへ戻す

Agentが暴走し得る
→ Stopping Condition / Guardrailで制御する
```

という考え方である。

Agentを人格として考えるのではない。

> **Environmentから情報を受け取り、LLMが次のActionを決め、Toolを使って実行し、その結果を見てまた判断するSoftware System**

として捉える。
