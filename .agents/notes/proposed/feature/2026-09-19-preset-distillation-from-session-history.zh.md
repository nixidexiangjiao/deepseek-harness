# Agent Note: 从会话历史蒸馏 agent preset

Status: proposed

[English](2026-09-19-preset-distillation-from-session-history.md) | 中文

## 问题

每个 preset 都是手写的，而"preset 里该写什么"的证据早已被记录下来，却从没有人读。

### preset 今天是怎么回事

**preset** 是一个 agent 会话所运行的那份配置。它是一个目录，其中的 `agent.cordis.yml`——即它的 **composition**——列出要挂载的插件：工具、prompt section 和 skill。[`dsh-agent-presets`](../../../../packages/preset/agent-presets/README.zh.md) 从三个根发现 preset，为每个 preset 挂载一份常驻 composition；composition 加载失败的 preset 会带着原因被列出，而不是被隐藏。

```text
one session ──runs under──► preset ──────► composition (agent.cordis.yml)
                            (a directory)        │
                                                 └── mounts plugins:
                                                     tools, prompt sections, skills
```

与此同时，每个会话都会写日志。[`dsh-session-query`](../../../../packages/session-query/session-query/README.zh.md) 已经能列出、过滤、读取和搜索这些日志，因此"哪些工具被调用、哪些 shell 命令反复出现、哪些文件被反复读取、用户在哪里纠正了 agent"这份记录，今天就能被应用代码取到。

但没有任何东西读这份记录，去回答 preset 所回答的那个问题：这个 agent 应该被配置成什么样。

### 显而易见的做法为什么被堵死

缺口不在于缺一个存储。而在于今天创建 preset 的唯一方式，是把一个现有 preset 整目录复制。

[authoring 模块](../../../../packages/preset/agent-presets/src/authoring.ts)只接受 preset id 和一个可选显示名，从不接受 composition 文本，并且写明了理由：authoring 不得授予被复制 preset 本身不携带的任何能力。能提供 composition 文本的调用方就能点名任意插件，所以一旦接受文本，preset authoring 就变成了"挂载任何东西"的途径。

于是个性化只能落在复制之后手改 `agent.cordis.yml`。没有任何记录能说明哪条观察支撑了哪一行，第二个人也无从审计。

## 提案

一个蒸馏器：读取会话历史，把它归约为计数，针对人工指定的**基准 preset** 提出一个 patch，并且只在人工接受后写入——全程受一条规则约束，使 authoring 的保证不被破坏。

### 五个阶段

五个阶段依次是 scan（扫描）、profile（画像）、propose（提议）、confirm（确认）、write（写入），只有最后一个会落盘。

```text
 ① scan              ② profile           ③ propose          ④ confirm         ⑤ write
 ─────────           ─────────           ─────────          ─────────         ───────
 session history     counts, not         a patch, plus      a human reads     copy(base, id)
 read through   ──►  transcripts:   ──►  the observations ──►  it and      ──► then apply
 dsh-session-query   tool histograms,    behind every       accepts or        the patch
                     recurring commands, proposed line      declines
                     hot files,               ▲                                    │
                     repeated corrections     │                                    ▼
                                         base preset,                        an ordinary
                                         named by the human                  preset directory
```

阶段 ⑤ 不产生任何特殊之物：结果由现有 roster 发现、挂载、列出和删除，不引入新的生命周期。

### 能力下界

这是整个设计的承重规则。蒸馏出的 preset 可以添加指令数据、可以收窄已有之物；但不得挂载其基准未挂载的插件。

```text
          base preset's mounted plugin set
          (computed by compositionInventory())
                        │
                        ▼
   ┌──────────────────────────────────────────────────┐
   │ the patch MAY                                    │
   │   add prompt sections    → dsh-persona           │──► written
   │   add file-backed skills → dsh-skill-filesystem  │
   │   narrow the config of a tool the base mounts    │
   ├──────────────────────────────────────────────────┤
   │ the patch MAY NOT                                │──► refused at
   │   mount any plugin the base does not mount       │    validation;
   └──────────────────────────────────────────────────┘    nothing is written
```

prompt section 和文件型 skill 能通过下界，是因为它们是指令文本而非能力授予：[`dsh-skill-filesystem`](../../../../packages/skill/skill-filesystem/README.zh.md) 把 skill 作为被扫描根下的文件来发现，[`dsh-persona`](../../../../packages/preset/persona/README.zh.md) 注册 prompt section。两者都不挂载任何新东西。

下界正是保住 authoring 保证的那一环。今天该保证成立，是因为 composition 文本从不来自调用方；在本提案下它成立，是因为生成文本在任何写入之前都要对照基准的已挂载插件集做检查，因此结果仍然不授予基准本就不携带的任何能力。

### 每部分住在哪里

| 部分 | 归属 | 理由 |
|---|---|---|
| 语料扫描、下界检查、写入 | 插件 | 下界是与安全相关的那一半，必须落在会 fail loud 的代码里。 |
| 哪个模式该变成 skill、收窄的工具，还是什么都不做 | 该包附带的一个 skill | 分类规则保持为可读可改的指令，而不被编译进 `src/`。 |

### 一个走通的例子

一位用户在同一个仓库上用 `standard` preset 工作了三个月。

扫描数出 412 次 `bash` 调用，其中 180 次以 `pytest` 开头；96 次读取 `conftest.py`；以及 14 个轮次，其下一条用户消息在纠正 agent 跑了整个测试套件而不是单个文件。

蒸馏器提议追加一个 prompt section，说明该仓库的测试默认按文件运行；再追加一个 skill，记录如何选定测试文件。它不提议任何工具改动，因为没有计数支撑。每一条提议都连同其背后的计数一并列出。

人工接受。写入是 `copy('standard', 'py-repo')` 再施加该 patch。结果中没有任何东西挂载了 `standard` 所没有的插件。

### 其余机制

- **先聚合再读取。** 画像由 `listSessions`、`filterEvents` 和 `searchEvents` 得到的计数构成。`readSession` 是留给少数计数无法解释的轨迹的选择性出口。任何一次运行都不会把整份会话日志放进模型请求。
- **提议与证据绑定。** 没有观察支撑的改动根本不会被提出。缺省字段继承基准，而继承永远是安全答案。
- **preset 文件仍然只是输入。** 挂载子树已经把 `write()` 覆盖为 no-op，这里没有任何东西会把 preset 变成持久化目标。
- **语料范围按次声明。** 默认只用本 harness 自己的存储。[hooks 桥接](../../../../packages/hooks/README.zh.md)能触及的其他存储按次选择加入，并写进产物，使读者能分辨哪段历史产生了哪一行。

训练数据抽取、路由信号、他人 preset 的远程安装、以及按计划自动重蒸馏，均不在范围内。

## 曾考虑的替代方案

- **让蒸馏器直接写 composition 文本：** 否决，因为这会抹掉 preset authoring 之所以狭窄的理由。能提供 composition 文本的调用方就能挂载任意插件，于是一个被自己语料误导的蒸馏器——transcript 是攻击者可触及的文本——就成了能力提升路径。下界把最坏情况限制在"基准本就能产生的文本"。
- **采用 RSIH 的十二组件所有权模型：** 否决。那套划分的价值来自扁平的 `settings.json`，字段所有权必须从外部强加。我们的配置是插件图，每个插件自己的 `Config` 已经划分了所有权，再加一套固定分类法只会和现有 package 分组竞争，且没有任何闸门维护它。
- **采用 RSIH 的 settings 编译：** 否决。启动时把声明的键改写进单一 settings 文件，会强制一个进程只有一个活跃画像。roster 为每个 preset 挂载一份常驻 composition 并把 agent scope 挂到其下，不同 preset 的会话本就能并发运行且状态分离；编译 settings 等于放弃这一点。
- **直接问用户想配什么：** 否决，因为这正是已经存在的路径。要读历史的理由恰恰是陈述的偏好与观察到的需要会分叉，而观察的那一侧是现有任何视图都不报告的。
- **整个蒸馏器只做成一个 skill，不要插件：** 否决，因为 skill 只能建议模型遵守下界，永远无法强制。

## 验收标准

- 蒸馏出的 preset 不挂载其基准 composition 中不存在的插件；会造成这种结果的 patch 在校验期被拒绝并点名越界插件，且不写入任何文件。
- 每一条提议都伴随支撑它的观察，确认产物把它们列出；没有观察支撑的字段在输出中缺席，而不是被赋默认值。
- 在不少于一千个会话的语料上运行，不发出任何包含整份会话日志的模型请求。
- 写出的 preset 原封不动地通过现有 roster 的发现与挂载路径，`remove()` 像删除任何本地 authoring preset 一样删除它。
- 在确认步骤选择拒绝，不留下 preset 目录，也不留下部分写入。
- 单元测试覆盖下界检查（接受 prompt、skill 或收窄型 patch；拒绝插件新增）、先聚合的扫描、以及 copy-then-patch 写入。由于产物与确认交互是模型可见的，一次端到端蒸馏由无密钥的录制会话快照覆盖。

## 风险

- **隐私残留是阻塞性前置条件，不是后续项。** 生成文本从真实 transcript 蒸出，必然携带绝对路径、内网主机名以及形似凭据的内容。RSIH 明知此缺口仍然发布并如实记录；我们不应如此。对生成文本的脱敏遍历、以及在确认步骤展示残留，属于首次落地的一部分。
- **下界依赖基准插件集可计算。** `compositionInventory()` 提供它，但基准 preset 可能按宿主门控行（随包发布的 `minimal` preset 就按 `process.platform` 禁用行），因此已挂载集是宿主相关的。下界在解析宿主上计算并在挂载时复查，清单无法解析的基准被拒绝而不是被近似。
- **没有闸门断言 preset 能表达宿主 composition 能表达的东西。** preset 承载工具、prompt section 和 skill；是否每个可配置字段都能从 preset 触及，当前无人检查，因此蒸馏器只能在恰好可表达的范围内提议。本提案不关闭该缺口，反而会让它显形；关闭它是一个独立的 `process` 类决策。
- **证据偏向近期和高强度使用。** 在无界窗口上构建画像，会让一个高强度的周定义整个 agent。聚合窗口、以及一个模式被计入前所需的最少出现次数，是可从 cordis.yml 修改的 `Config` 字段而非常量，遵循"插件中不得有硬编码可调项"的规则。
- **蒸馏出的 preset 仍然是受信配置。** roster 本就要求把每个 authoring preset 当作受信配置对待，因为它授予其所选插件的能力。下界收窄了蒸馏能添加的东西，但并不使结果免于审阅，而确认步骤就是那次审阅。
