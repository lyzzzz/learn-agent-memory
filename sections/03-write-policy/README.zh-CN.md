# 3 · 写入策略（Write policy）

[English](README.md) · [繁體中文](README.zh-TW.md) · **简体中文**

> 一条信息能不能成为记忆，需要明确判断。每个决定都要说明理由，并留下记录。

这是[生产环境中的记忆系统](../../README.zh-CN.md)教程的第三章。原始事件采集之后，要经过写入审核（write gate），才能决定是否把其中的信息保存为记忆。送来审核的内容叫候选记忆（candidate）：它从新事件中整理而来，正在等待写入决定。

存得太多，检索时就会混入大量噪声和过期信息，成本也会增加；存得太少，agent 又会反复询问用户已经回答过的问题。如果这些决定没有记录，出了问题就很难解释“为什么记住了这个，却忘了那个”。

基础实现通常把筛选规则写在提取器的提示词里：只保存四类信息，而且必须是无法从其他地方重新查出或推导出来的事实。到了生产环境，还需要进一步做到：

1. 按明确的标准分别判断候选记忆，不能只笼统地问“重不重要”。
2. 区分不同的处理结果，除了保存和忽略，还要支持暂缓处理、等待批准。
3. 写入前验证每个决定，无论它来自规则还是模型。
4. 保存每个决定的理由，包括拒绝写入的理由。

---

## 实现机制

最小实现可以是一个函数：接收候选记忆，返回处理决定。关键在于检查顺序。先运行成本低、结果确定的规则，规则无法判断时再调用模型。最后，由验证函数检查决定是否符合要求，通过后才能执行。

这里有五个部分：

- **Candidate：** 待保存的记忆，来自新写入的原始事件，并带有来源事件 ID。
- **规则：** 按成本安排顺序，检查有没有证据、能否重新推导、是否重复、是否具体，以及是否敏感。
- **分类器（classifier）：** 通过一次模型调用处理规则难以判断的情况，例如这条信息以后还用得上，还是一句闲聊。
- **Decision：** 保存动作、理由和置信度。动作有四种：`store`（保存）、`ignore`（忽略）、`defer`（暂缓）和 `require_approval`（等待批准）。
- **验证器（validator）：** 用确定的规则检查决定，避免不合格的结果直接写入。

六个判断标准分别由这些部分负责：

```text
Novelty       是否有新信息：用规则比较它与已有记忆的词语重叠程度
Specificity   是否足够具体：过于模糊的信息，以后很难检索到
Derivability  能否重新查出：用 grep 或 git 就能找到的信息，不必另存
Sensitivity   是否敏感：带敏感标记的信息需要人工批准
Durability    是否长期有用：交给分类器区分有用信息与闲聊
Confidence    判断有多可靠：由做出决定的一方填写，随决定一起保存
```

候选记忆和决定的数据结构都很简单：

```python
@dataclass(frozen=True)
class Candidate:
    content: str
    kind: str                          # episodic / semantic / procedural
    source_event_ids: tuple = ()
    sensitive: bool = False

@dataclass(frozen=True)
class Decision:
    action: str                        # store / ignore / defer / require_approval
    reason: str
    confidence: float
```

`decide` 按顺序执行规则。一旦某条规则给出决定，就直接返回；所有规则都无法判断时，才交给分类器。如果没有接入分类器，则默认保存：

```python
def decide(candidate, existing=(), classifier=None) -> Decision:
    for rule in RULES:                 # evidence, derivability, duplicate, vagueness, sensitivity
        if decision := rule(candidate, existing):
            return decision
    if classifier is not None:
        return Decision(*classifier(candidate))
    return Decision(STORE, "novel, specific, evidence-backed", 0.6)
```

重复检查有两个阈值。词语重叠程度很高，说明很可能已经记过，直接忽略；重叠程度中等，则先暂缓。相似记忆的合并由第 6 章处理，不在写入审核阶段完成：

```python
def _duplicate(c, existing):
    best = max((_overlap(c.content, m) for m in existing), default=0.0)
    if best >= DUPLICATE_AT:
        return Decision(IGNORE, "already known", 0.9)
    if best >= SIMILAR_AT:
        return Decision(DEFER, "similar memory exists, merge at consolidation", 0.6)
```

不管决定来自哪里，都必须经过 `validate`。模型可以提出建议，但只有验证通过的决定才能执行：

```python
def validate(decision, candidate) -> Decision:
    if decision.action not in ACTIONS:
        raise ValueError(f"unknown action: {decision.action}")
    if not decision.reason:
        raise ValueError("a decision without a reason cannot be audited")
    if not 0 <= decision.confidence <= 1:
        raise ValueError("confidence out of range")
    if decision.action == STORE and not candidate.source_event_ids:
        raise ValueError("store without source events")
    return decision
```

`gate` 把这些步骤连接起来，同时记录决定本身：

```text
新写入的事件（第 2 章）
    ↓ 整理出候选记忆
规则检查：证据 · 能否重新推导 · 重复程度 · 是否模糊 · 是否敏感
    ↓ 只有规则无法判断的内容，才交给分类器
分类器：保存 / 忽略 / 暂缓 / 等待批准
    ↓ 检查每个决定
确定性的验证器
    ↓ 执行通过验证的决定
生成分类记忆（第 4 章）· 把决定作为事件写入日志
```

这一阶段处理第 2 章新采集的事件，在任务结束后运行，也就是温路径。决定保存的候选内容会变成第 4 章的分类记录；暂缓的内容交给第 6 章合并整理；需要批准的则进入审核队列，和 Hermes 暂存写入、等待审批的做法类似。

每个决定也会写回事件日志。第 10 章会利用这些可回放的记录，评估写入的准确程度。

还有一些研究尝试直接训练写入策略：[Memory-R1](https://arxiv.org/html/2508.19828v2) 根据任务结果，用强化学习学习选择 `ADD / UPDATE / DELETE / NOOP`；[AgeMem](https://arxiv.org/abs/2601.01885) 则进一步把记忆操作纳入 agent 自身的行为策略。这两种方法都需要针对具体任务的训练数据，不适合作为第一版实现。

### 比基础流程多了什么

以前，筛选逻辑藏在提取器的提示词里，外部只能看到文件写入了，或者什么也没发生。现在，每次判断都有明确的结果和理由：相似记忆先暂缓，敏感信息等待批准，被拒绝的候选内容也能在日志里查到原因。

---

## 各系统的做法

| | Mem0 | Hermes Agent |
| --- | --- | --- |
| **优点** | 写入时也处理更新，新事实可以直接替换旧事实。 | 敏感写入可以等待人工批准；会话过程中就能保存，减少结束时遗漏信息的风险。 |
| **局限** | 每次写入都需要调用模型；更新或删除判断错误时，会改动原有数据。 | 审批需要人工参与，队列可能积压；判断规则仍写在提示词里。 |
| **设计考虑** | 保持记忆库精简，让新事实融入已有内容。 | 模型负责提议，启用审批后由人决定最终是否写入。 |
| **候选内容从哪里来** | 模型从最新的对话中提取。 | 模型认为信息以后仍有用时，在会话中调用记忆工具。 |
| **怎样决定** | 对照相似的已有记忆，选择新增、更新、删除或不操作。 | 默认直接写入；开启审批模式后，先进入待审状态。 |
| **如何回查或把关** | 每条记忆保留操作历史，出错后可以回查。 | 逐条列出待审写入，供人批准或退回。 |

---

## 常见问题

- **审核太严格。** 记忆库里存不下多少信息，agent 反复询问已知内容。应按规则分别跟踪忽略比例；对难以确定的候选内容，可以先暂缓，留给合并整理阶段再判断。
- **审核太宽松。** 检索结果中的噪声会越来越多。第 10 章的写入精确率和重复率可以反映这个问题。先收紧重复与具体程度的要求，再考虑调整分类器。
- **拒绝决定没有记录。** 用户问“为什么不记得这件事”时，无法追查原因。所有决定都要记录理由，包括忽略。
- **直接使用分类器输出。** 无法识别的动作、超出范围的置信度都应明确报错。每个决定必须通过验证，才能执行。
- **完全让模型判断敏感信息。** 模型一旦漏看标记，秘密就可能被保存。应先检查敏感标记、匹配模式和来源渠道等确定性信号，模型只提供补充判断。
- **审核拖慢查询。** 每轮对话都同步评分，会增加响应时间。应在任务结束后批量处理候选记忆。

---

## 可运行的代码

[`src/`](src/) 在第 2 章的基础上增加：

- [`policy.py`](src/policy.py)：`Candidate`、`Decision`、规则链，以及 `decide`、`validate` 和 `gate`。
- [`engine.py`](src/engine.py)：`propose()` 把候选内容送去审核，并把每个决定写成事件。
- [`test.py`](src/test.py)：分别触发各条规则，检查分类器是否只在规则无法判断时调用、验证器能否拒绝格式错误的决定，以及决定是否写入事件日志。

```bash
python sections/03-write-policy/src/test.py   # offline checks, no key
```

离线检查使用模拟的分类器。接入真实模型后，每条规则无法判断的候选记忆会调用一次模型。

---

## 参考来源

- [Mem0](https://arxiv.org/abs/2504.19413)：先提取再更新，对照相似记忆选择新增、更新、删除或不操作。
- [Memory-R1](https://arxiv.org/html/2508.19828v2)：用强化学习训练写入策略。
- [AgeMem](https://arxiv.org/abs/2601.01885)：把记忆操作纳入 agent 的行为策略。
- [Hermes Agent 源码](https://github.com/NousResearch/hermes-agent)：`tools/write_approval.py`，暂存写入并等待批准。
- [生产环境中的记忆系统](../../README.zh-CN.md)：本章在整个记忆处理流程中的位置。
