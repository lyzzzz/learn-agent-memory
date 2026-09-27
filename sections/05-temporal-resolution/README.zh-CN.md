# 5 · 处理时间与冲突（Temporal resolution）

[English](README.md) · [繁體中文](README.zh-TW.md) · **简体中文**

> 一条信息现在不成立，不代表它以前是错的。应该标明它何时失效，并保留当时的记录。

这是[生产环境中的记忆系统](../../README.zh-CN.md)教程的第五章，负责解析实体身份、处理信息冲突，以及记录时间变化。

第 4 章让每条记忆有了类型和出处，但还没解决一个问题：新记录和旧记录说法不一致时，该怎么办？

直接覆盖会丢掉历史，无法追查系统上个月认为事实是什么、为什么后来改变了判断。如果两条都保留为有效状态，检索又会返回互相矛盾的结果，让模型自行猜测。

处理这类情况，要分别回答三个问题：

1. **是否指向同一个对象？** “Marcus”和“Ming-Siang”说的是不是同一个人？
2. **是否真的矛盾？** “住在 San Diego”和“住在 San Francisco”，是说法冲突，还是分别在不同时间成立？
3. **各自在什么时候成立？** 哪条描述现在的情况，哪条描述过去的情况？

---

## 实现机制

最小实现会为每条记录保存两组时间，并保留旧记录：

```text
recorded_at ─────────── superseded_at    系统采用这条信息的期间：从记下到被替换
valid_from  ─────────── valid_to         这条信息在现实中成立的期间
```

这叫双时间模型（bitemporal）。两组时间分别回答不同的问题：“系统当时认为 Marcus 住在哪里”，看第一组；“Marcus 三月实际住在哪里”，看第二组。

比如，今天才补记了一件去年的事。只记去年，就看不出系统今天才知道；只记今天，又丢掉了事情发生的时间。两组时间都保留，才能表达这种情况。[Graphiti 和 Zep](https://arxiv.org/abs/2501.13956) 的时间知识图谱也采用了类似设计：旧关系会被标记为失效，但不会从图中消失。

判断冲突之前，还要知道哪些记录在描述同一个问题。每条记录因此带有一个主张键（claim key）：用“主语 + 谓语”标识所讨论的属性，用内容表达具体取值。例如，两条记录都在描述 Marcus 的住址：

```text
claim_key "marcus:lives_in"   content "Marcus lives in San Diego"      status superseded  valid_to 2025-06
claim_key "marcus:lives_in"   content "Marcus lives in San Francisco"  status active      valid_from 2025-06
```

同一个键下，如果内容不同、两条又都处于有效状态，就需要处理冲突，确认哪条已经失效。没有填写键的记录不参与这种自动替换。

这里的操作都保留历史。和第 2 章的事件日志一样，不提供任意改写内容的 `UPDATE` 或直接删除记录的 `DELETE`，而是使用四种明确的操作：

```text
ADD        新增一条有效记录
SUPERSEDE  让旧记录失效，由新记录接替
RETRACT    标记记录本来就是错的，但仍保留供回查
ABSTRACT   从多条记录中归纳出一条更概括的记录，第 6 章会用到
```

更新事实时，先新增记录，再用 `SUPERSEDE` 标记旧记录失效。严格说，这也修改了旧记录，但修改范围受到限制：只允许填写 `status`、`superseded_at`、`valid_to` 这些结束状态字段，单向修改一次。原有内容保持不变，之后仍然可以读取。

`supersede` 和 `retract` 要分清：前者表示“以前成立，现在变了”，后者表示“这条信息原本就是错的”，常见原因是提取错误。撤回的记录也需要保留，因为它能帮助定位提取逻辑的问题。

写入流程因此多出一步。`resolve` 为验证通过的记录确定时间状态，并让与它冲突的旧记录失效：

```python
def resolve(store, record, at=None) -> list:
    ops = []
    for old in conflicts(record, same_scope(record.scope, store.records)):
        closed, op = supersede(old, at, cause_id=record.id)
        store.records[store.records.index(old)] = closed
        ops.append(op)
    store.add(replace(record, valid_from=record.valid_from or at))
    return ops + [Operation(ADD, record.id, "new claim", at)]
```

写入时也必须检查归属范围。替换旧记录属于写入操作，因此 A 租户的新信息不能影响 B 租户的同名记录。`same_scope` 会先排除其他范围的记录，再进行冲突判断。

两组时间对应两种查询：

```python
def as_of(records, when):     # what the system believed at a past moment
    return [r for r in records if r.recorded_at <= when
            and (r.superseded_at is None or r.superseded_at > when)]

def valid_at(records, when):  # what was true in the world at that moment
    return [r for r in records if (r.valid_from is None or r.valid_from <= when)
            and (r.valid_to is None or r.valid_to > when)]
```

第 3 章审核通过的候选内容，在第 4 章生成分类记录后，进入本章处理时间与冲突。处理函数返回操作列表，引擎再把每个操作写成事件，因此这些时间变化也能被重新回放。

第 8 章只检索当前有效的记忆；第 9 章则读取记录之间的冲突关系，在组装上下文时列出双方信息，避免在没有说明的情况下只选一边。

### 接入前面代码时发现的两个问题

下面两个问题都来自第 3 章的写入审核，但直到本章真正接入更正流程，才在第一版实现中暴露出来。修正保留在本章的代码中，前面章节仍保留各自当时的版本，方便对比整合时需要调整什么。

**问题一：更正被当成疑似重复内容。** 原来的审核规则按词语重叠程度判断相似性，过于相似的候选记忆会被暂缓处理。但更正往往与原句十分接近，例如“Marcus lives in San Diego”和“Marcus lives in San Francisco”。

如果更正被暂缓，冲突处理就收不到新记录，旧住址会一直保持有效。这恰好阻止了本章需要处理的信息进入系统。

解决办法是：候选内容带有主张键时，只把文字完全相同的内容判为重复；其他情况作为可能的更正，交给冲突处理阶段判断：

```python
def _duplicate(c, existing):
    if c.claim_key:
        # 带 claim key 的 candidate 不量重叠：字面完全相同才算重复，
        # 其他一律当成更正，交给 resolution 判断
        return Decision(IGNORE, "already known", 0.9) if c.content in existing else None
    best = max((_overlap(c.content, m) for m in existing), default=0.0)
    if best >= DUPLICATE_AT:
        return Decision(IGNORE, "already known", 0.9)
    if best >= SIMILAR_AT:
        return Decision(DEFER, "similar memory exists, merge at consolidation", 0.6)
```

两句话可能只差一个词，意思却完全不同，词语重叠比例无法可靠地区分这种情况。

**问题二：敏感信息检查排得太晚。** 审核规则按顺序运行，遇到第一个决定就返回。原来敏感检查排在最后，候选内容一旦先被判为疑似重复，就不会再接受敏感检查。

这类安全检查必须先于可能让内容进入存储的规则执行。调整后的顺序如下：

```python
RULES = (_no_evidence, _sensitive, _derivable, _duplicate, _vague)
```

### 本章增加了什么

第 4 章的记录创建后基本不再变化。现在，记录有了有效期，可以被更正和替换。原来只是随记录保存的 `status` 字段，也开始由替换操作实际更新。

---

## 各系统的做法

| | Graphiti / Zep | Claude Code 自动记忆 |
| --- | --- | --- |
| **优点** | 可以查询旧事实，并说明对应日期；写入时就处理矛盾。 | 不需要复杂的时间建模，更正时直接修改文件，用户也可以还原。 |
| **局限** | 每条关系都要保存两组时间，数据结构更复杂，提取时也必须准确识别时间。 | 文件本身不保留旧内容，改写后原来的说法就消失了。 |
| **设计考虑** | 记忆会不断被更正，因此把“旧信息失效”直接纳入数据模型。 | 假设用户会查看记忆目录，把文件本身作为检查记录的入口。 |
| **处理冲突** | 在图中新增关系，同时让与之矛盾的旧关系失效。 | 由模型直接修改相关记忆文件。 |
| **记录时间** | 使用双时间模型，同时保存事件时间和系统写入时间。 | 只有文件当前状态所隐含的时间，没有两组显式时间字段。 |
| **恢复方式** | 从完整保留的对话经历中重建图。 | 取决于用户是否为记忆目录另行配置了版本控制。 |

---

## 常见问题

- **直接覆盖旧内容。** 历史信息消失后，既无法解释为什么更新，也难以恢复错误修改。替换操作应补充结束时间和状态，不删除原记录。
- **冲突记录同时有效。** 检索会返回互相矛盾的说法。应在写入时处理冲突，让同一个主张键下只保留一条当前有效记录。
- **把错误记录按正常过期处理。** 对提取错误的记录使用 `supersede`，会让它看起来像“以前成立过”，影响之后的 `valid_at` 查询。这种情况应使用 `retract`。
- **把更正挡在重复检查里。** 更正与原文可能只差一个词。带主张键时，应改用完全相同的文字判断重复，再把其他情况交给冲突处理。
- **敏感检查太晚。** 前面的规则可能已经返回决定，导致检查被跳过。敏感检查必须放在可能保存内容的规则之前。
- **替换影响了其他租户。** 两个租户可能保存相同的句子，但不能互相替换记录。先检查归属范围，再判断冲突。
- **同一属性使用不同的主张键。** `marcus:lives_in` 和 `marcus:location` 不会被识别为同一个问题，两条矛盾记录就会同时保留。键名应来自统一维护的词表。
- **没有先确认实体身份。** “Marcus”和“Ming-Siang”如果代表同一个人，却使用不同的键，冲突就无法被发现。应先解析身份，再比较记录。

---

## 可运行的代码

[`src/`](src/) 在第 4 章的基础上增加：

- [`resolve.py`](src/resolve.py)：操作类型、`conflicts`、`supersede`、`retract`、`resolve`，以及分别查询两组时间的 `as_of` 和 `valid_at`。
- [`records.py`](src/records.py)：双时间字段、`claim_key`，以及拒绝不合理时间关系的验证。
- [`policy.py`](src/policy.py)：带主张键时只按文字完全相同判断重复；提前检查敏感信息；`words` 会过滤停用词，避免仅因为含有“the”就命中。
- [`engine.py`](src/engine.py)：处理记录的时间状态，把操作写回日志，并通过 `believed_at` 和 `true_at` 分别查询系统认知时间与现实有效时间。
- [`test.py`](src/test.py)：检查冲突与范围隔离、替换与撤回的区别、补记事件时两组时间的差异、不合理时间关系是否被拒绝，以及一次完整更正能否让旧记录失效。

```bash
python sections/05-temporal-resolution/src/test.py   # offline checks, no key
```

本章不调用模型，因此没有 `demo.py`。

---

## 参考来源

- [Zep / Graphiti](https://arxiv.org/abs/2501.13956)：双时间知识图谱，旧关系失效后仍然保留。
- [A-Mem](https://arxiv.org/html/2502.12110v1)：相互关联的笔记会随着新记忆加入而更新。
- [生产环境中的记忆系统](../../README.zh-CN.md)：本章在整个记忆处理流程中的位置。
