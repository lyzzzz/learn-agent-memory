<h1 align="center" style="margin-top: 0;">Learn Agent Memory</h1>

<p align="center">
  <strong>看看实际运行的 agent 是怎样记住信息的</strong><br>
</p>

<p align="center">
  <a href="#各章节"><img src="https://img.shields.io/badge/Focus-Memory_Engineering-8250df" alt="Focus: Memory Engineering"></a>
  <a href="#研究的系统"><img src="https://img.shields.io/badge/Systems-10-0969da" alt="Systems"></a>
  <a href="#各章节"><img src="https://img.shields.io/badge/Sections-10-2da44e" alt="Sections"></a>
  <a href="LICENSE"><img src="https://img.shields.io/badge/License-MIT-d29922" alt="License"></a>
</p>

<p align="center">
  <a href="README.md">English</a> · <a href="README.zh-TW.md">繁體中文</a> · <strong>简体中文</strong>
</p>

给 agent 配上向量数据库，只是记忆系统的一部分。真正用于生产环境的系统，还要完整保存原始证据，再从证据中整理出便于查询的记忆。这样，即使整理过程出了错，也能重新生成。

多数 agent 的基础做法差不多：用文件保存记忆，用索引查找；每轮对话开始前找出相关记忆，一次任务结束后提取新信息，同时保留原始会话日志。只有一个用户、一个 agent 时，这套流程就够用了。

但在生产环境中，系统可能要同时服务多个租户、多个 agent，还要处理积累了好几年的历史记录。记忆因此成了一个独立的子系统：信息什么时候写入、什么时候更新、出了问题怎么恢复，都需要专门设计。这个仓库会用十章内容，把基础流程逐步扩展成完整的记忆系统，每章解决一个设计问题。

**目录：** [记忆的处理流程](#记忆的处理流程) · [学习方法](#学习方法) ·
[研究的系统](#研究的系统) · [各章节](#各章节) · [文件结构](#文件结构) · [运行检查](#运行检查)

---

## 记忆的处理流程

![生产环境中的记忆处理流程](assets/production-memory.png)

整套设计遵循一个基本原则：

> 保留原始事件，保证整理出来的记忆随时可以重建。

采集之后的每一层都是“视图”：从原始事件生成、方便查询的一份数据，而不是信息的唯一副本。提取、合并或索引出了问题，都可以从事件日志重新生成。
[MemMachine](https://arxiv.org/abs/2604.04853) 也采用了这个思路：完整保存每段对话经历（episode），作为核对信息的依据；用户画像、索引和带上下文的检索都建立在这些原始记录之上。

### 哪些工作什么时候做

系统里的工作分三个时机执行：查询时立即处理、任务结束后处理，以及放到后台处理。它们分别叫热路径（hot path）、温路径（warm path）和冷路径（cold path）。

| | 热路径：查询时 | 温路径：任务结束后 | 冷路径：后台处理 |
| --- | --- | --- | --- |
| **执行时机** | 每次查询 | 一次任务运行结束时 | 后台运行或定时执行 |
| **负责的工作** | 制定检索计划、检索、重新排序、组装上下文、把内容放入提示词 | 追加原始事件、审核写入、提取候选记忆 | 合并整理、去重、替换过期记忆、更新用户画像、重建索引、评估 |
| **限制** | 响应要快，并严格控制 token 用量 | 可以多调用一次模型，但不能长时间阻塞 | 可以慢一些，但必须保证操作安全 |

基础流程其实已经有了这三个时机：对话前检索，任务结束后提取，后台合并整理。本教程会沿用这个安排，逐步补齐每个阶段需要的能力。

---

## 学习方法

每章都可以独立阅读，也都包含以下四部分：

1. **要解决的问题：** 这一章为什么有必要。
2. **实现机制：** 需要哪些组件，数据怎样在它们之间流转。
3. **各系统的做法：** 用表格对比实际系统的实现。
4. **常见问题：** 哪些地方容易出错，怎么避免或处理。

建议这样阅读和练习：

- **按章节顺序读。** 代码会在上一章的基础上逐步增加功能。
- 运行每章的离线检查：`python sections/NN-name/src/test.py`。不需要 API 密钥。
- 对比相邻两章的 `src/` 目录。两者的代码差异，就是这一章新增的机制。

---

## 研究的系统

下面这些系统会出现在相应章节的对比表中，帮助你理解同一个问题有哪些不同的解决办法。

| 系统 | 大家为什么用它 | 值得关注的设计 | 涉及章节 |
| --- | --- | --- | --- |
| **[Claude Code](https://docs.claude.com/en/docs/claude-code/memory)** | 当前编程能力最强的 agent，自动记忆功能按项目目录保存 Markdown 文件。 | 按范围隔离存储、后台合并整理 | 1、5 |
| **[Hermes Agent](https://github.com/NousResearch/hermes-agent)** | 适合长期使用的助理：记住用户、学习工作流程，还能跨平台执行任务。 | 原始会话日志、写入审批 | 1、2、3 |
| **[MemMachine](https://arxiv.org/abs/2604.04853)** | 开源记忆层，保留完整对话经历，作为原始证据。 | 对话记录、带上下文的检索 | 2、8 |
| **[Mem0](https://arxiv.org/abs/2504.19413)** | 使用广泛的记忆层，会把新事实合入已有记忆，避免记录不断堆积。 | 由大语言模型判断如何写入 | 3 |
| **[LangMem](https://github.com/langchain-ai/langmem)** | LangChain 的记忆 SDK，写入时按应用定义的数据结构校验记录。 | 分类记录、用户画像 | 4 |
| **[Hindsight](https://arxiv.org/html/2512.12818v1)** | 把事实、观察和意见分开保存的记忆引擎。 | 区分信息依据、通过反思修订记忆 | 4 |
| **[Graphiti / Zep](https://arxiv.org/abs/2501.13956)** | 用时间知识图谱保存记忆。旧事实会被标为失效，同时保留历史。 | 两组时间字段、`SUPERSEDE` 操作 | 5 |
| **[A-Mem](https://arxiv.org/html/2502.12110v1)** | 由 agent 自主整理记忆：新笔记会关联旧笔记，并更新相关内容。 | 动态建立链接、自主合并整理 | 6 |
| **[Sleep-time Compute](https://arxiv.org/html/2504.13171v1)** | 把记忆整理放到后台，减少查询时的等待。 | 在冷路径中整理记忆 | 6 |
| **[AgentRunbook-C](https://arxiv.org/abs/2605.12493)** | 把执行过程存成文件，让编程 agent 在沙箱里自行搜索。 | 由 agent 自主检索文件 | 8 |

> 第 7 到 10 章侧重比较设计方式，包括 wiki 与图视图、检索策略、上下文组装规则和评估指标，不围绕某一个系统展开。

---

## 各章节

全书共十章。每章讨论一个设计问题，附有可以独立阅读的说明和可运行的代码。

| # | 章节 | 要回答的问题 | 关键机制 |
| --- | --- | --- | --- |
| | **提取记忆** | | |
| 1 | [记忆系统的接口约定](sections/01-memory-contract/README.zh-CN.md) | 这份记忆属于谁？ | 数据范围、租户与用户隔离、保留策略、敏感程度 |
| 2 | [原始事件记录](sections/02-event-ledger/README.zh-CN.md) | 什么能作为证据？ | 只追加的日志、事件发生时间与系统记录时间 |
| 3 | [写入策略](sections/03-write-policy/README.zh-CN.md) | 哪些信息值得记住？ | 信息是否新增、是否长期有用、明确记录写入决定 |
| 4 | [记忆的分类](sections/04-typed-memory/README.zh-CN.md) | 这条记忆有什么用途、依据是什么？ | 经历、知识、操作流程，以及信息依据的分类 |
| | **整理记忆** | | |
| 5 | [处理时间与冲突](sections/05-temporal-resolution/README.zh-CN.md) | 信息矛盾了，还是情况变了？ | 双时间模型、`SUPERSEDE`、保留历史的操作 |
| 6 | [合并与整理](sections/06-consolidation/README.zh-CN.md) | 怎样从事件中整理出知识？ | 压缩、归纳、提出方案后验证再执行 |
| 7 | [索引与视图](sections/07-index-views/README.zh-CN.md) | 同一份事件记录怎样支持多种查询？ | 稀疏索引、向量索引、时间、图、wiki 和用户画像视图 |
| | **检索并使用记忆** | | |
| 8 | [混合检索](sections/08-hybrid-retrieval/README.zh-CN.md) | 怎样找到需要的记忆？ | BM25、向量与图检索，补充来源上下文，选择检索方式 |
| 9 | [组装上下文](sections/09-context-assembly/README.zh-CN.md) | 怎样把记忆安全地交给模型？ | 完整的证据信息、token 预算、明确标为不可信的参考数据 |
| 10 | [评估与管理](sections/10-evaluation-governance/README.zh-CN.md) | 记忆到底有没有帮上忙？ | 写入、检索、上下文和端到端效果的评估指标 |

---

## 文件结构

```text
learn-agent-memory/
├── README.md                      # 项目总览
├── sections/                      # 每章一个文件夹
│   ├── 01-memory-contract/        # 从这里开始逐章实现，每章都有 README.md
│   ├── ...
│   └── 10-evaluation-governance/  # 完整的记忆引擎
└── assets/                        # 共用图片
```

章节目录统一命名为 `NN-name/`。每个目录都有英文、繁体中文、简体中文三份 README，以及可以运行的 `src/` 代码。
每章保留上一章的代码，再增加一个机制。因此，对比相邻两章就能看出新增功能；第 10 章包含完整的记忆引擎。

---

## 运行检查

代码只使用 Python 标准库，例如 `dataclasses` 和 `sqlite3`，没有第三方依赖。不需要 API 密钥，也不需要额外安装包。

每章都有一个 `test.py`，用于离线检查。从仓库根目录运行即可，例如：

```bash
python sections/01-memory-contract/src/test.py
```

---

## 参与贡献

- **补充系统实例。** 在相关章节的对比表中加入其他记忆系统。
- **完善章节。** 补充实现机制、改进图示，或更准确地分析出错原因。
- **修正内容。** 这些教学说明是根据论文和文档整理的，欢迎附上出处来纠正问题。

请尽量使用已有名称、能够查证的机制，并注明来源，避免只凭猜测描述实现。

---

## 参考资料

- [MemMachine](https://arxiv.org/abs/2604.04853)：保留原始证据，检索时补充命中内容所在对话的完整上下文。
- [Zep / Graphiti](https://arxiv.org/abs/2501.13956)：用时间知识图谱保存 agent 记忆，每条事实记录两种时间。
- [Hindsight](https://arxiv.org/html/2512.12818v1)：通过保留、检索、反思三个步骤，把事实、观察和意见分开管理。
- [A-Mem](https://arxiv.org/html/2502.12110v1)：由 agent 自主整理记忆，新笔记会关联并更新旧笔记。
- [Memory-R1](https://arxiv.org/html/2508.19828v2)：用强化学习训练记忆管理器，让它学习选择 `ADD / UPDATE / DELETE / NOOP`。
- [Sleep-time Compute](https://arxiv.org/html/2504.13171v1)：利用没有查询的空闲时间预先整理，减少查询时的计算成本。
- [Karpathy 的 LLM Wiki](https://gist.github.com/karpathy/442a6bf555914893e9891c11519de94f)：原始构想文件。由大语言模型维护 Markdown wiki，保留原始来源，页面内容都从来源中整理。
- [HippoRAG](https://arxiv.org/abs/2405.14831)：结合知识图谱和个性化 PageRank，一次检索找齐需要跨多层关系关联的证据。
- [LongMemEval](https://arxiv.org/abs/2410.10813)：评估长期交互记忆的基准，包含五类任务。
- [LongMemEval-V2](https://arxiv.org/abs/2605.12493)：把评估范围扩展到 agent 的工作经验，AgentRunbook-C 的文件检索方法也来自这项工作。
