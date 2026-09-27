# 2 · 原始事件记录（Event ledger）

[English](README.md) · [繁體中文](README.zh-TW.md) · **简体中文**

> 原始观察写入后就保留下来，不再修改。后续整理出来的数据，都应该能从这些记录中重建。

这是[生产环境中的记忆系统](../../README.zh-CN.md)教程的第二章，负责采集原始数据。我们会建立一份只追加、不修改的事件日志，后续各章都从这里读取原始信息。

提取记忆时，模型难免会判断失误或漏掉细节。如果系统只保存提炼后的结论，出错后就没有原始记录可供核对。

最简单的做法是保留会话日志：运行结束后，把整轮对话存进 SQLite，之后按关键词搜索。但生产环境中的事件记录（ledger）还需要满足几个条件：

1. 能接收不同类型的原始事件，不只保存聊天内容。
2. 每条记录都标明数据归属。
3. 分别记录事件发生的时间，以及系统获知事件的时间。
4. 由数据库阻止修改和删除，而不是只靠开发者遵守约定。

---

## 实现机制

最小实现是一张只能追加的 SQLite 表，写入时只允许使用 `INSERT`。需要更正，就追加一条新记录；需要让某条记录不再出现在查询结果中，就在下游视图里排除它，原始记录仍然保留。

这里有四个主要部分：

- **Scope：** 标明租户、用户和 agent。写入时填写，读取时用来过滤。
- **Event：** 一条不可变的观察记录，包含 ID、类型、内容、两种时间戳和元数据。
- **Ledger：** 提供 `append` 和 `read` 两个操作，分别负责追加与读取。
- **两个数据库触发器：** 直接阻止 SQLite 执行 `UPDATE` 和 `DELETE`，即使调用方写错了代码，也不能改掉原始数据。

记录类型用普通的 dataclass 定义。教程中的设计示意可以用 Pydantic 模型表达，可运行版本只依赖标准库：

```python
@dataclass(frozen=True)
class Scope:
    tenant_id: str
    user_id: str | None = None
    agent_id: str | None = None

@dataclass(frozen=True)
class Event:
    id: str
    scope: Scope
    event_type: str
    content: str
    occurred_at: str | None    # true in the world since; None when unknown
    recorded_at: str           # known to the system since; always set
    metadata: dict
```

“不可修改”由数据库结构保证，不依赖代码审查来兜底：

```sql
CREATE TRIGGER IF NOT EXISTS events_no_update BEFORE UPDATE ON events
    BEGIN SELECT RAISE(ABORT, 'ledger is append-only'); END;
CREATE TRIGGER IF NOT EXISTS events_no_delete BEFORE DELETE ON events
    BEGIN SELECT RAISE(ABORT, 'ledger is append-only'); END;
```

`append` 直接执行一条 `INSERT`，不需要先读取再写入。`recorded_at` 由事件日志自己填写，因此调用方不能把“系统获知这件事的时间”倒填成更早的日期：

```python
def append(self, scope, event_type, content, occurred_at=None, metadata=None) -> str:
    event_id = uuid.uuid4().hex
    con = self._db()
    con.execute("INSERT INTO events VALUES (?, ?, ?, ?, ?, ?, ?, ?, ?)",
                (event_id, scope.tenant_id, scope.user_id, scope.agent_id,
                 event_type, content, occurred_at, _now(), json.dumps(metadata or {})))
    con.commit()
    con.close()
    return event_id
```

`read` 的每个查询都必须带上租户条件。其他租户的数据从查询阶段就被排除，不会交给调用方。数据隔离必须先得到保证，不能指望检索时再处理：

```python
def read(self, scope, event_type=None, since=None) -> list[Event]:
    sql, args = "SELECT * FROM events WHERE tenant_id = ?", [scope.tenant_id]
    for column, value in (("user_id", scope.user_id), ("agent_id", scope.agent_id),
                          ("event_type", event_type)):
        if value is not None:
            sql += f" AND {column} = ?"
            args.append(value)
    ...
```

数据按下面的顺序流转：

```text
用户对话 · 工具结果 · 更正 · 批准 · 任务结果
        ↓ append：只执行 INSERT
events 表：归属范围、两种时间戳、数据库强制只追加
        ↓ read：必须按租户过滤
写入审核（第 3 章）· 合并整理（第 6 章）· 重建索引（第 7 章）
```

agent 运行框架会在固定时机调用 `append`：一轮对话结束、工具返回结果、用户更正信息，以及任务完成时。
后续主要有三类读取者：写入审核读取新事件，判断该记住什么；合并整理读取历史，处理合并与替换；索引出错时，重读这些事件来恢复数据。

### 比普通会话日志多了什么

- 会话日志按会话区分数据，事件日志则按归属范围区分。多个租户可以共用一张表，同时保持数据隔离。
- `event_type` 让日志能够保存聊天之外的证据，例如工具结果、更正、批准和任务成败。
- `occurred_at` 记录事件发生时间，`recorded_at` 记录系统获知时间，第 5 章会用它们处理时间问题。
- 只追加不修改的约定由触发器强制执行，对所有调用方都生效。
- 每条事件都有独立 ID。提炼出的记忆通过 `source_event_ids` 引用这些 ID，就能找到原始证据。

---

## 各系统的做法

MemMachine 和 Hermes 都在整理后的记忆之外保留原始历史，主要区别是记录单位：前者保存完整的一段对话经历，后者把每条消息存成一行。

| | MemMachine | Hermes Agent |
| --- | --- | --- |
| **优点** | 用户画像或索引出错后，可以从完整对话重建。命中一条内容时，还能展开前后文。 | 在现有数据库中加一张表即可，不需要新服务；无需调用模型就能搜索历史消息。 |
| **局限** | 为每个用户保存完整对话占用较多空间，需要考虑归档。 | 只保存聊天消息，没有事件类型、归属范围字段和事件发生时间。 |
| **设计考虑** | 整理后的各层数据都可以重新生成，完整对话才是原始依据。 | 提取过程可能遗漏信息，因此保留原始历史供回查。 |
| **记录单位** | 一整段对话经历，内容完整保留。 | 每条消息一行，记录会话 ID、角色和文字。 |
| **写入时机** | 对话数据到达时就写入。 | 运行结束时，按会话批量写入。 |
| **读取方式** | 先找到命中内容，再返回它在对话中的前后文。 | 关键词搜索加模糊排序，相似度高的结果排在前面。 |
| **衍生数据** | 用户画像和索引都能从完整对话重建。 | 另外维护两份 Markdown 记忆文件，日志保留它们背后的原始信息。 |

---

## 常见问题

- **日志不断增长。** 读取会变慢，存储成本也会上升。可以按归属范围和时间分区，把较少使用的历史迁到对象存储，但归档后仍要能够读取，才能用于重建。不要在没有记录的情况下直接清掉旧数据。
- **敏感信息长期留存。** 只追加的规则会与用户删除请求发生冲突。应在采集时标明敏感程度，法律要求的删除则通过专门、可审计的清除流程完成。这是唯一允许破坏原始数据的例外，而且要记录删除了什么。
- **猜测事件发生时间。** 填错 `occurred_at` 会影响第 5 章的时间推理。不知道就留空。`recorded_at` 由日志自行填写，才是系统能够确认的时间。
- **把提炼的结论当成原始证据。** 摘要一旦混入原始事件，后面就可能一直被当成证据引用。原始日志只收观察到的内容；整理后的记忆放在下游，通过事件 ID 指回来源。
- **多个进程同时写入。** 并发追加可能遇到数据库锁。追加操作应保持为一条 `INSERT`，避免先读后写；多进程写入时开启 WAL 模式。

---

## 可运行的代码

[`src/`](src/) 在第 1 章的基础上增加：

- [`ledger.py`](src/ledger.py)：`Event`、`Ledger.append`、`Ledger.read`，以及两个保证只追加的触发器。
- [`engine.py`](src/engine.py)：开始整合各个组件，`observe()` 会向事件日志追加一条记录。
- [`test.py`](src/test.py)：检查范围隔离、两种时间戳、数据库是否阻止修改、视图能否按范围分别重建，以及 `observe()` 是否确实写入了事件日志。

```bash
python sections/02-event-ledger/src/test.py   # offline checks, no key
```

本章不调用模型，因此没有 `demo.py`。

---

## 参考来源

- [MemMachine](https://arxiv.org/abs/2604.04853)：以完整对话作为原始依据，各层衍生数据都能据此重建。
- [Hermes Agent 源码](https://github.com/NousResearch/hermes-agent)：`hermes_state.py` 中的 `SessionDB`，本章在这类会话日志的思路上做了扩展。
- [生产环境中的记忆系统](../../README.zh-CN.md)：本章在整个记忆处理流程中的位置。
