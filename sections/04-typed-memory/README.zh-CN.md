# 4 · 记忆的分类（Typed memory）

[English](README.md) · [繁體中文](README.zh-TW.md) · **简体中文**

> 保存一条记忆时，要说明它能用来回答什么问题，也要说明系统凭什么知道这件事。

本章对应[生产环境中的记忆系统](../../README.zh-CN.md)中的记录编码阶段：把第 3 章审核通过的候选记忆，转成有分类、经过验证的正式记录。

“上周二部署失败了”“Marcus 主要写 Python”“执行数据库结构迁移前，先检查迁移锁”，都是一段文字，但用途并不一样。它们回答不同的问题，过时的方式不同，检索时也需要区别处理。如果存储时完全不分类，后面就很难用好这些信息。

另外，亲自观察到的内容和模型推测的内容也必须分开。否则，系统迟早会把猜测当成事实说出来。

文件式记忆常按用户、反馈、项目、参考资料这四类保存，重点是判断“哪些东西值得留下”。本章则从使用方式出发：

1. 按用途分类：记录发生过的事、当前掌握的知识，或下次可以采用的做法。
2. 另加一组“信息依据”标签：原始观察、已确认事实、用户偏好、系统推论或主观意见。
3. 创建记录时就验证，拒绝缺少证据或类型不合法的内容。
4. 保留来源，让每条记忆都能追溯到原始事件。

---

## 实现机制

最小实现包括两组枚举值，以及一个带验证的构造函数。

第一组是用途类型，在代码中叫 `kind`：

```text
episodic     经历：发生过什么。
             例如：“部署被迁移锁卡住，最后通过回滚解决。”
             保留时间、环境和结果。事情过去了，它仍然是一段真实的历史。

semantic     知识：目前认为成立的信息。
             例如：“Marcus 主要写 Python。”
             可以从经历中提炼，也可以根据新证据修订。

procedural   操作流程：下次遇到类似情况该怎么做。
             例如：“执行数据库结构迁移前，先检查迁移锁。”
             包括工作流程、操作手册和需要留意的问题。
```

第二组是信息依据，在代码中叫 `epistemic_type`。它和用途类型是两个独立维度：前者回答“凭什么知道”，后者回答“拿来做什么”：

```text
evidence     原始证据：直接来自事件日志的观察
fact         已确认事实：经过验证，或由用户亲口说明
preference   用户偏好：记录用户想要什么，不把偏好当成客观事实
inference    系统推论：系统推测的结论，可以修订，也必须明确标注
opinion      主观意见：保留意见的身份，不能在没有说明的情况下变成事实
```

一条记录同时包含用途、信息依据和来源。所有记录统一通过 `make_record` 创建，并在返回前完成验证：

```python
@dataclass(frozen=True)
class MemoryRecord:
    id: str
    scope: Scope
    kind: str                  # episodic / semantic / procedural
    epistemic_type: str        # evidence / fact / preference / inference / opinion
    content: str
    source_event_ids: tuple    # the evidence, from section 2
    confidence: float
    recorded_at: str
    status: str = "active"     # active / superseded / retracted
    tags: tuple = ()

def make_record(scope, kind, epistemic_type, content, source_event_ids,
                confidence, tags=()) -> MemoryRecord:
    return validate(MemoryRecord(...))
```

验证方式与第 3 章相同：用固定规则检查，发现问题就抛出异常。没有来源事件的记录不能通过，因为系统无法给出它的证据：

```python
def validate(record) -> MemoryRecord:
    if record.kind not in KINDS:
        raise ValueError(f"unknown kind: {record.kind}")
    ...
    if not record.source_event_ids:
        raise ValueError("a record without source events cannot cite its evidence")
```

存储时按 `kind` 分类，读取时按归属范围和状态过滤。`grounded` 只选出原始证据与已确认事实，避免把推论混在其中：

```python
def current(self, scope, kind) -> list[MemoryRecord]:
    """Active records of one kind for one scope, newest first."""

def grounded(records) -> list[MemoryRecord]:
    return [r for r in records if r.epistemic_type in ("evidence", "fact")]
```

数据从第 3 章进入这里后，会补上用途类型和信息依据，成为正式记录。第 5 章再添加两组时间字段，分别说明内容在什么时候成立、系统在什么时候记录，并在替换旧记录时更新 `status`。

第 8 章检索时会按用途选择查询方式；第 9 章把记忆放进上下文时，会把信息依据标在内容旁边，让模型区分观察与推论。

### 分类方式变在哪里

用户、反馈、项目、参考资料这四类文件，主要回答“什么值得留下”。本章的两组标签分别回答“拿来做什么”和“凭什么知道”。是否值得保存，已经由第 3 章决定；这里负责把保存下来的内容组织成清楚的记录。

[Hindsight](https://arxiv.org/html/2512.12818v1) 也强调类似的区分：实际发生的事情，要与 agent 对这些事情的看法分开。

---

## 各系统的做法

| | LangMem | Hindsight |
| --- | --- | --- |
| **优点** | 写入时按应用自定义的数据结构验证。 | 把实际经历与 agent 的看法分开；看法可以修订，经历保留不动。 |
| **局限** | 类型能发挥多大作用，取决于应用的数据结构设计。 | 分成四个库后，分类工作更多；分错类型就会存入错误的库。 |
| **设计考虑** | 不同应用需要不同的记忆结构，因此由应用定义结构。 | 只有区分观察与意见，才能避免把猜测当成事实。 |
| **分类方式** | 知识（semantic）、经历（episodic）、操作流程（procedural）。 | 世界事实（world fact）、经历（experience）、观察（observation）、意见（opinion）四个库。 |
| **记录单位** | 每个用户一份画像文件，或一组符合指定结构的记录。 | 各库中的分类条目，由反思流程写入。 |
| **更新方式** | 直接修订画像，或在记录集合中新增、更新条目。 | 反思流程读取新证据、修订看法，并引用所依据的内容。 |

---

## 常见问题

- **所有内容都归为知识。** 这样分类就失去了作用。写入时应区分：带时间和结果的事件属于经历，带适用条件的操作说明属于流程。
- **把推论标成事实。** 模型的猜测可能因此被当成事实传播。创建记录时必须填写信息依据，第 9 章组装上下文时也要显示这个标签。
- **有分类，没有来源。** 没有证据就无法核查、替换或信任这条记录。`source_event_ids` 为空时，一律拒绝。
- **数据结构不断变化。** 新增字段或类型后，旧记录可能不再符合要求。由于这些记录都从第 2 章的原始事件派生，可以重新处理事件来生成新结构，不必逐条迁移旧记录。
- **操作说明丢了适用条件。** 只保存“检查迁移锁”，却漏掉“执行数据库结构迁移前”，系统就不知道什么时候该使用它。流程类记忆要同时保留条件和操作。

---

## 可运行的代码

[`src/`](src/) 在第 3 章的基础上增加：

- [`records.py`](src/records.py)：两组枚举值、`MemoryRecord`、`make_record`、`validate`、`grounded` 和 `TypedStore`。
- [`engine.py`](src/engine.py)：把审核通过的候选记忆转成经过验证的分类记录。至此，`observe()` 对应的写入流程已包含事件记录、写入审核和正式记忆。
- [`test.py`](src/test.py)：检查错误类型、不合法置信度和缺少来源的记录是否被拒绝；验证分类读取、范围隔离、`grounded` 筛选与状态过滤；确认引擎保存的记录沿用审核决定中的置信度。

```bash
python sections/04-typed-memory/src/test.py   # offline checks, no key
```

本章不调用模型，因此没有 `demo.py`。

---

## 参考来源

- [Hindsight](https://arxiv.org/html/2512.12818v1)：区分事实、经历、观察和意见。
- [LangMem](https://github.com/langchain-ai/langmem)：按指定数据结构验证记忆，提供用户画像与记录集合。
- [生产环境中的记忆系统](../../README.zh-CN.md)：本章在整个记忆处理流程中的位置。
