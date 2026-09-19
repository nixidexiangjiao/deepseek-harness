# Agent Note: 从会话历史蒸馏 agent preset

Status: proposed

[English](2026-09-19-preset-distillation-from-session-history.md) | 中文

## 问题

一句话：agent 的配置是手写的，而它已经跑过的那些会话早就记录了这份配置该写什么——本文提议把这份记录读出来，据此提出一份改动，并且只在人工接受之后才写入。

### 本文用到的术语

| 术语 | 在本文中的含义 |
|---|---|
| DSH | DeepSeek Harness，即本仓库。每个包都命名为 `@deepseek-ai/dsh-<name>`。 |
| session（会话） | 用户与 agent 之间一次从头到尾的对话。每个会话都会写一份日志。 |
| session log（会话日志） | 一次会话的持久记录：每一次模型请求、工具调用、工具结果和用户消息。 |
| transcript（文字记录） | 会话日志里的文本内容。它包含用户和工具产生的任何东西，因此不是可信输入。 |
| plugin（插件） | 一个可以被开启的行为单元。一个工具、一个 prompt section、一个 skill 提供方，各自都是插件。 |
| mount（挂载） | 为某个 agent 开启一个插件，使其注册生效。 |
| preset | 一个 agent 会话所运行的那份配置。在磁盘上它是一个目录。 |
| composition | preset 目录里的 `agent.cordis.yml` 文件，列出该 preset 要挂载哪些插件。 |
| roster（名册） | [`dsh-agent-presets`](../../../../packages/preset/agent-presets/README.zh.md) 所发现、挂载、并可删除的那份 preset 列表。 |
| authoring（创建） | 通过 roster 创建或删除 preset，区别于手工改文件。 |
| prompt section | 加进 agent 系统提示词的一段文本，由 [`dsh-persona`](../../../../packages/preset/persona/README.zh.md) 注册。 |
| skill | 一个装着任务专用指令的文件，agent 可按需加载，由 [`dsh-skill-filesystem`](../../../../packages/skill/skill-filesystem/README.zh.md) 发现。 |
| base preset（基准 preset） | 一次蒸馏所基于的那个现有 preset。由人工指定，蒸馏器从不自行挑选。 |
| patch | 本提案要施加在基准 preset 之上的那组改动。 |
| capability floor（能力下界） | 限制 patch 内容的那条规则，定义见下文《能力下界》一节。 |
| RSIH | RSI-Harness，另一个项目，为另一个 agent 解决同样的配置问题。仅在下文《曾考虑的替代方案》一节中被引用。 |

### 本文提到的函数

| 调用 | 归属 | 作用 |
|---|---|---|
| `listSessions()`、`filterEvents()`、`searchEvents()` | [`dsh-session-query`](../../../../packages/session-query/session-query/README.zh.md) | 读取会话历史而不加载整份日志。 |
| `readSession()` | `dsh-session-query` | 完整读取一份会话日志。 |
| `copy(from, id, name)` | `dsh-agent-presets` | 把一个现有 preset 目录复制为一个新 id。这是今天唯一能创建 preset 的写入。 |
| `remove(id)` | `dsh-agent-presets` | 删除一个本地创建的 preset。 |
| `compositionInventory()` | `dsh-agent-presets` | 报告每个 preset 的 composition 挂载了哪些插件。 |

### preset 今天是怎么回事

```text
   session  ────►  preset  ────►  agent.cordis.yml  ────►  plugin  plugin  plugin
      ①              ②                   ③                        ④
```

| 标号 | 元素 | 说明 |
|---|---|---|
| ① | 会话 | 恰好运行于一个 preset 之下，或显式指定，或取自配置的默认值。 |
| ② | preset | 一个目录。roster 从三个根之一找到它，并为每个 preset 挂载一份常驻 composition。 |
| ③ | composition | 列出要挂载的插件。composition 加载失败的 preset 会带着原因被列出，而不是被隐藏。 |
| ④ | 插件 | 工具、prompt section 和 skill。agent 能做什么，恰好就等于这里挂载了什么。 |

每个会话同时也会写日志，而 `dsh-session-query` 已经能列出、过滤、读取和搜索这些日志。因此"哪些工具被调用、哪些 shell 命令反复出现、哪些文件被反复读取、用户在哪里纠正了 agent"这份记录，今天就能被应用代码取到。

但没有任何东西读这份记录，去回答 preset 所回答的那个问题：这个 agent 应该被配置成什么样。

### 显而易见的做法为什么被堵死

缺口不在于缺一个存储。而在于创建 preset 的唯一方式是 `copy()`——把一个现有 preset 整目录复制。

[authoring 模块](../../../../packages/preset/agent-presets/src/authoring.ts)只接受 preset id 和一个可选显示名，从不接受 composition 文本，并且写明了理由：authoring 不得授予被复制 preset 本身不携带的任何能力。能提供 composition 文本的调用方就能点名任意插件，所以一旦接受文本，preset authoring 就变成了"挂载任何东西"的途径。

于是个性化只能落在复制之后手改 `agent.cordis.yml`。没有任何记录能说明哪条观察支撑了哪一行，第二个人也无从审计。

## 提案

一个蒸馏器：读取会话历史，把它归约为计数，针对人工指定的基准 preset 提出一份 patch，并且只在人工接受后写入——全程受一条规则即能力下界约束，使 authoring 的保证不被破坏。

### 五个阶段

```text
   ┌────┐    ┌────┐    ┌────┐    ┌────┐    ┌────┐
   │ ①  │───►│ ②  │───►│ ③  │───►│ ④  │───►│ ⑤  │
   └────┘    └────┘    └────┘    └────┘    └────┘
                                    │
                                    └───► ✗ ───► ∅
```

| 阶段 | 名称 | 发生什么 | 读还是写 |
|---|---|---|---|
| ① | 扫描 | 经 `dsh-session-query` 读取会话历史。 | 读日志 |
| ② | 画像 | 归约为计数：工具调用直方图、反复出现的 shell 命令、热点文件、重复的用户纠正。 | 仅内存 |
| ③ | 提议 | 针对基准 preset 构造一份 patch，并附上每一条提议背后的观察。 | 仅内存 |
| ④ | 确认 | 人工阅读 patch 及其证据，然后接受或拒绝。 | 不动任何东西 |
| ⑤ | 写入 | 执行 `copy(base, id)`，再施加该 patch。 | 写磁盘 |
| ✗ | 拒绝 | 人工在 ④ 选择拒绝。 | 不动任何东西 |
| ∅ | — | 不留下 preset 目录，也不留下任何半截文件。 | — |

只有阶段 ⑤ 会落盘，而它产出的是一个普通 preset：由现有 roster 发现、挂载、列出和删除，不引入新的生命周期。

### 能力下界

这是整个设计的承重规则：蒸馏出的 preset 可以追加指令文本、可以收窄已有之物，但不得挂载其基准未挂载的插件。

| patch 想做的事 | 是否允许 | 经由 | 结果 |
|---|---|---|---|
| 追加一个 prompt section | 允许 | `dsh-persona` 的 prefix 与 suffix | 写入 |
| 追加一个文件型 skill | 允许 | 该 preset 自带的 `skills/` 目录 | 写入 |
| 收窄某个已挂载工具自身的限额 | 允许 | 该工具的 `Config` | 写入 |
| 改动其他任何配置字段 | **不允许** | — | 校验期拒绝并点名该字段 |
| 挂载一个基准未挂载的插件 | **不允许** | — | 校验期拒绝并点名该插件，不写入任何文件 |

基准的已挂载插件集来自 `compositionInventory()`，因此"基准是否挂载了这个"是算出来的答案，而不是一次判断。

被允许的配置改动是一份**显式字段白名单**，而不是"已挂载插件的任意字段"。一个配置字段同样能扩大行为，其程度不亚于新增一行，因此下界点名它允许的三处：`dsh-persona` 的 `prefix` 与 `suffix`、用于注册该 preset 自带 `skills/` 目录的那条 `customSkillDirs`、以及对某个已挂载工具自身限额的收窄改动。

prompt section 和文件型 skill 能通过下界，是因为它们是指令文本而非能力授予：skill 是一个 agent 可以去读的文件，prompt section 是加进系统提示词的文本。两者都不挂载任何新东西。

下界正是保住 authoring 保证的那一环。今天该保证成立，是因为 composition 文本从不来自调用方；在本提案下它成立，是因为生成文本在任何写入之前都要对照基准的已挂载插件集做检查，因此结果仍然不授予基准本就不携带的任何能力。

### 每部分住在哪里

| 部分 | 归属 | 理由 |
|---|---|---|
| 语料扫描、下界检查、写入 | 插件 | 下界是与安全相关的那一半，必须落在会 fail loud 的代码里。 |
| 哪个复现模式该变成 skill、收窄的工具，还是什么都不做 | 该包附带的一个 skill | 分类规则保持为可读可改的指令，而不被编译进 `src/`。 |

### 一个走通的例子

一位用户在同一个 Python 仓库上用随包发布的 `standard` preset 工作了三个月。该 preset 本就挂载了 `dsh-persona` 和 `dsh-skill-filesystem`，这正是下面这份 patch 合法的原因。

阶段 ① 与 ② 读入 143 个会话并把它们归约为计数。阶段 ③ 把这些计数变成下面这份确认产物——它就是阶段 ④ 给人看的全部内容：

```text
preset distillation · base: standard · new id: py-repo
corpus: own store, 143 sessions, 2026-06-19 … 2026-09-19

[1] persona.suffix                                    +1 sentence
      Tests in this repository are run per file by default. Run the
      whole suite only when asked.
    evidence  180 of 412 bash calls begin `pytest`
              14 user turns correct a whole-suite run

[2] skills/select-test-target/SKILL.md                 new file
    evidence  96 reads of conftest.py across 61 sessions
              11 sessions re-derive the same file-selection steps

[3] skill-filesystem.customSkillDirs                   +1 entry
      <preset>/skills/
    reason    required by [2]; allowlisted field, plugin already mounted

not proposed: tools, model, runtime, policies, appearance
              no observation reached the configured minimum of 8

capability floor  OK · 0 plugins added · 3 allowlisted fields touched
                  accept / decline ?
```

人工接受，阶段 ⑤ 执行 `copy('standard', 'py-repo')` 再施加该 patch，产出这样一个目录：

```text
~/.dsh/.agent-presets/py-repo/
├── preset.yml
├── agent.cordis.yml
└── skills/
    └── select-test-target/
        └── SKILL.md
```

被复制的 `agent.cordis.yml` 里有两行发生改动，且两者都属于 `standard` 本就挂载的插件：

```diff
 - id: persona
   name: '@deepseek-ai/dsh-persona'
   config:
-    suffix: Your working directory is {{cwd}}.
+    suffix: >-
+      Your working directory is {{cwd}}.
+      Tests in this repository are run per file by default. Run the whole
+      suite only when asked.
     prefix: >-
       You are a coding agent powered by the {{model}} model.

 - id: skill-filesystem
   name: '@deepseek-ai/dsh-skill-filesystem'
+  config:
+    customSkillDirs:
+      - !!js "process.getBuiltinModule('node:url').fileURLToPath(new URL('skills/', baseUrl))"
```

唯一新增的文件是一个普通的 skill bundle：

```markdown
---
name: select-test-target
description: Use when running tests in this repository, to choose which test file to run instead of the whole suite.
---

# Selecting a test target

Run one file with `pytest <path>`; run the whole suite only when asked.

To find the file for a change, locate the nearest `conftest.py` and the test module importing the changed module.
```

假如同一份语料还产生了一条"挂载某个插件"的提议——七个会话在问插件内部机制，可能指向 `cordis` preset 挂载而 `standard` 没有的 `@deepseek-ai/dsh-tool-cordis`——下界会拒绝它，且该次运行什么都不写：

```text
[4] + plugin row '@deepseek-ai/dsh-tool-cordis'
    evidence  7 sessions ask about plugin internals

capability floor  REFUSED
  '@deepseek-ai/dsh-tool-cordis' is not mounted by base preset 'standard'
  nothing was written
```

蒸馏解决不了这个诉求；人工要么换一个本就挂载该插件的基准 preset，要么放弃。

### 其余机制

- **先聚合再读取。** 画像由 `listSessions`、`filterEvents` 和 `searchEvents` 得到的计数构成。`readSession` 是留给少数计数无法解释的轨迹的选择性出口。任何一次运行都不会把整份会话日志放进模型请求。
- **提议与证据绑定。** 没有观察支撑的改动根本不会被提出。缺省字段继承基准，而继承永远是安全答案。
- **preset 文件仍然只是输入。** 挂载子树已经把 `write()` 覆盖为 no-op，这里没有任何东西会把 preset 变成持久化目标。
- **语料范围按次声明。** 默认只用本 harness 自己的存储。[hooks 桥接](../../../../packages/hooks/README.zh.md)能触及的其他存储按次选择加入，并写进产物，使读者能分辨哪段历史产生了哪一行。

训练数据抽取、路由信号、他人 preset 的远程安装、以及按计划自动重蒸馏，均不在范围内。

## 曾考虑的替代方案

- **让蒸馏器直接写 composition 文本：** 否决，因为这会抹掉 preset authoring 之所以狭窄的理由。能提供 composition 文本的调用方就能挂载任意插件，于是一个被自己语料误导的蒸馏器——transcript 是攻击者可触及的文本——就成了能力提升路径。下界把最坏情况限制在"基准本就能产生的文本"。
- **采用 RSIH 的十二组件所有权模型：** 否决。那套划分的价值来自扁平的 `settings.json`，字段所有权必须从外部强加。DSH 的配置是插件图，每个插件自己的 `Config` 已经划分了所有权，再加一套固定分类法只会和现有 package 分组竞争，且没有任何闸门维护它。
- **采用 RSIH 的 settings 编译：** 否决。启动时把声明的键改写进单一 settings 文件，会强制一个进程只有一个活跃画像。roster 为每个 preset 挂载一份常驻 composition 并把 agent scope 挂到其下，不同 preset 的会话本就能并发运行且状态分离；编译 settings 等于放弃这一点。
- **直接问用户想配什么：** 否决，因为这正是已经存在的路径。要读历史的理由恰恰是陈述的偏好与观察到的需要会分叉，而观察的那一侧是现有任何视图都不报告的。
- **整个蒸馏器只做成一个 skill，不要插件：** 否决，因为 skill 只能建议模型遵守下界，永远无法强制。

## 验收标准

- 蒸馏出的 preset 不挂载其基准 composition 中不存在的插件；会造成这种结果的 patch 在校验期被拒绝并点名越界插件，且不写入任何文件。
- 每一条提议都伴随支撑它的观察，确认产物把它们列出；没有观察支撑的字段在输出中缺席，而不是被赋默认值。
- 在不少于一千个会话的语料上运行，不发出任何包含整份会话日志的模型请求。
- 写出的 preset 原封不动地通过现有 roster 的发现与挂载路径，`remove()` 像删除任何本地创建的 preset 一样删除它。
- 在阶段 ④ 选择拒绝，不留下 preset 目录，也不留下部分写入。
- 单元测试覆盖下界检查（接受 prompt、skill 或收窄型 patch；拒绝插件新增）、先聚合的扫描、以及 copy-then-patch 写入。由于产物与确认交互是模型可见的，一次端到端蒸馏由无密钥的录制会话快照覆盖。

## 风险

- **隐私残留是阻塞性前置条件，不是后续项。** 生成文本从真实 transcript 蒸出，必然携带绝对路径、内网主机名以及形似凭据的内容。RSIH 明知此缺口仍然发布并如实记录；DSH 不应如此。对生成文本的脱敏遍历、以及在阶段 ④ 展示残留，属于首次落地的一部分。
- **下界依赖基准插件集可计算。** `compositionInventory()` 提供它，但基准 preset 可能按宿主门控行（随包发布的 `minimal` preset 就按 `process.platform` 禁用行），因此已挂载集是宿主相关的。下界在解析宿主上计算并在挂载时复查，清单无法解析的基准被拒绝而不是被近似。
- **没有闸门断言 preset 能表达宿主 composition 能表达的东西。** preset 承载工具、prompt section 和 skill；是否每个可配置字段都能从 preset 触及，当前无人检查，因此蒸馏器只能在恰好可表达的范围内提议。本提案不关闭该缺口，反而会让它显形；关闭它是一个独立的 `process` 类决策。
- **证据偏向近期和高强度使用。** 在无界窗口上构建画像，会让一个高强度的周定义整个 agent。聚合窗口、以及一个模式被计入前所需的最少出现次数，是可从 cordis.yml 修改的 `Config` 字段而非常量，遵循"插件中不得有硬编码可调项"的规则。
- **蒸馏出的 preset 仍然是受信配置。** roster 本就要求把每个创建出的 preset 当作受信配置对待，因为它授予其所选插件的能力。下界收窄了蒸馏能添加的东西，但并不使结果免于审阅，而阶段 ④ 就是那次审阅。
