# Agentic System 5つの基本Workflow

## 1. Prompt Chaining

**フロー別・工程別の分業。**

```text
LLM A
↓ Output
LLM B
↓ Output
LLM C
```

1つの複雑な仕事を、固定された複数工程に分けます。

例えば、

```text
課題検索
↓
実装案作成
↓
スライド作成
```

のような構造です。

成果物の形や処理の責務が変わるところが、分離境界になりやすいです。

1つの成果物でも、

```text
コード
↓
リファクタリング
↓
命名規則確認
↓
誤字チェック
```

のように、処理責務を分ける意味があればChainingできます。

途中には `gate` を置けます。

```text
LLM
↓
中間成果物
↓
Gate
├─ OK → 次工程
└─ NG → Retry / Stop
```

**前の工程の結果を待つ必要があるならChaining。**

---

## 2. Routing

**職能別分掌に近い。**

```text
Input
↓
Router
├─ 技術相談 → Technical
├─ 返金要求 → Refund
└─ 一般質問 → General
```

Prompt Chainingが、

```text
企画 → 実装 → レビュー
```

というフロー別分業なのに対して、Routingは、

```text
営業
技術
経理
法務
```

のような専門領域への振り分けです。

巨大な1つのPromptですべて処理するのではなく、それぞれに特化したPrompt・Workflow・Toolへ流します。

```text
Prompt Chaining
= 工程別分業

Routing
= 職能別分掌
```

---

## 3. Parallelization

**独立した仕事を同時に処理する。**

### Sectioning

1つの仕事を、独立した観点に分けます。

例えば、

```text
「このコードをレビューして」

        ┌→ セキュリティ
コード ─┼→ 可読性
        ├→ 性能
        └→ 命名規則
                ↓
               統合
```

つまり、

> 「このコードをレビューして」を、「セキュリティ観点」「可読性」「性能」「命名規則」でレビューして、とPromptを分ける。

ということです。

**待つ必要があるならChaining、待たなくてよいならSectioning。**

命令パイプラインとは少し違い、同じ入力に対する処理同士がそもそも独立しています。

### Voting

**ダブルチェックや統計的確認に近い。**

```text
同じ問題
├→ LLM A
├→ LLM B
├→ LLM C
└→ LLM D
      ↓
     集約
```

同じ問題を複数回解かせて、多数決、一致度、閾値などで判断します。

```text
Sectioning
= 違う観点を並列化

Voting
= 同じ問題を複数回試行
```

Parallelizationは、分割方法が事前に決まっている**定型分業**に向いています。

職務分掌やチェック項目が固定されている、いわば「JTC型」の仕事とも相性がよいです。

---

## 4. Orchestrator-Workers

**将軍・足軽型。**

```text
User
↓
Orchestrator（将軍）
↓
その場でタスクを分解
├→ Worker A（足軽）
├→ Worker B
├→ Worker C
└→ Worker D
↓
結果を統合
```

Parallelizationとの最大の違いは、

```text
Parallelization
= 人間が事前に分割方法を決める

Orchestrator-Workers
= Orchestratorが入力を見て分割方法を決める
```

ことです。

つまり、Orchestratorが噛むことで**一段と自律性が高くなる**。

例えばコーディングなら、

```text
「認証機能を追加して」
↓
Orchestrator
↓
今回は
- DB変更
- API変更
- Frontend変更
- Test追加
が必要
```

と、毎回必要な仕事自体を動的に決められます。

検索でも、

```text
何を検索するか
どの観点に分割するか
どの情報源を見るか
追加調査が必要か
```

が検索してみるまで分からないため、Orchestratorが有効です。

不可分なのではなく、

> **分割可能だが、どう分割すべきかを事前には固定できない。**

というタスク向けです。

```text
Parallelization
= 定型分業

Orchestrator-Workers
= 非定型分業
```

---

## 5. Evaluator-Optimizer

**成果物のレビュワーと改善ループ。**

```text
Generator
↓
Solution
↓
Evaluator
├─ Accepted → Out
└─ Rejected + Feedback
        ↓
     Generator
        ↓
     再評価
```

Evaluatorは、

```text
この成果物で目的を達成しているか
何が不足しているか
どこを改善すべきか
```

を評価します。

例えば検索なら、

```text
検索・分析
↓
Evaluator
「まだ根拠が足りない」
↓
追加検索
↓
再分析
```

となります。

文学翻訳なら、

```text
翻訳
↓
Evaluator
「このニュアンスを捉えられていない」
↓
修正版を生成
```

となります。

`Optimizer` は必ず独立したLLMである必要はありません。

```text
Feedbackを受けたGenerator自身が改善
```

でも、

```text
問題のあるWorkerだけRetry
```

でも、

```text
検索工程まで戻す
```

でも、

```text
Orchestratorに再計画させる
```

でも構いません。

つまり、

```text
Evaluator
= 成果物のレビュワー

Optimizer
= Feedbackを受けて改善する役割
```

です。

Generator自体も、

```text
Chaining
Routing
Parallelization
Orchestrator-Workers
```

などで構築された**Agentic System全体を1つのでかい箱として見たもの**でも構いません。

そのため、Evaluatorの判定次第で、**構築された構造的Agentic Systemそのものを必要な地点からやり直す**こともできます。

---

## 5つを一言で整理

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

これらは独立したテクニックというより、**組み合わせてAgentic Systemを構築する基本的な制御構造**として捉える。

```text
予測可能な部分
→ Workflowで固定

不確定な部分
→ LLM / Orchestratorに判断させる

独立処理
→ Parallelization

品質確認
→ Evaluator

問題あり
→ 必要な工程へ戻してRetry
```

全部をAgentに任せるのではなく、**決め打ちできるところはWorkflowにして、判断が必要なところだけAgentにする。**
