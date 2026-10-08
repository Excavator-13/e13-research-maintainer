# Research Maintainer

一个让科研项目的状态可以跨 agent 会话恢复的 **Agent Skill**。

它的目标是让新接手的 agent 不依赖历史聊天记录，仅凭文件和证据就能回答五个问题：

1. 研究目标是什么？
2. 当前子任务做到哪里？
3. 下一步可执行动作是什么？
4. 哪些建议尚未决定？
5. 产物保存、入库、备份到了哪一步？

## 解决什么问题

长周期科研工作通常由多个 agent 会话接力完成，常见失效方式是：

- 进度只存在于上一次对话的总结里，会话结束后无法恢复；
- 跑成功一次就被当成任务完成、阶段完成；
- 新发现直接改写旧协议，历史证据失去可追溯性；
- "已提交"、"已推送"、"已备份"被混为一谈；
- 封存摘要反过来要求记录包含它自己的 commit 或哈希，形成自指循环。

本 skill 用一套**职责分离的权威记录**来处理这些问题：每类事实只有一个权威位置，其他文档通过 ID 引用，不再维护相互竞争的进度表。

## 仓库结构

```text
research-maintainer/
  SKILL.md                      # 入口：定位、规则、动作选择、核验情景
  agents/openai.yaml            # Codex 界面元数据（显示名与默认 prompt）
  references/
    records.md                  # 标准目录、权威记录划分、各类记录模板
    transitions.md              # 接手、部分完成、新发现、收尾、停止规则、首次规范化
    archive-and-git.md          # 按请求：封存、Git 入库、推送与异地备份
```

## 资料路由

skill 采用渐进式披露，只读当前动作需要的参考文件：

| 当前动作 | 阅读 |
| --- | --- |
| 初始化 / 规范化 / 新增记录 | [`references/records.md`](research-maintainer/references/records.md) |
| 接手 / 部分完成 / 中断 / 新建议 / 阶段变更 | [`references/transitions.md`](research-maintainer/references/transitions.md) |
| 用户明确要求封存 / 提交 / 推送 / 恢复核验 | [`references/archive-and-git.md`](research-maintainer/references/archive-and-git.md) |

## 记录结构

规范化后，项目记录被组织为以下标准结构（无相关内容时不建立空目录）：

```text
RESEARCH.md                       # 稳定入口，链接 state / roadmap / 归档索引
research/
  roadmap.md                      # 研究问题、假设、阶段目标、进入/退出条件
  state.md                        # 当前阶段与任务指针、恢复位置、下一动作
  decisions.md                    # 建议来源、接受情况、决定与计划变更理由
  phases/P01/
    plan.md                       # 阶段状态、任务状态、依赖、验收、计划修订
    protocol.md                   # 需冻结的实验定义（按需）
  runs/<run-id>/
    summary.md                    # 实际过程、结果与证据、解释与限制
    provenance.json               # 运行代码身份、协议、配置、环境、输入清单
    commands.log                  # 精确命令
    scripts/ inputs/ outputs/
  archives/
    index.md                      # 保存状态、SHA256、入库依据、备份核验
    <run-id>.tar.gz(+.sha256)     # 包外校验，不做自指哈希
    migrations/<id>.md            # 旧路径到标准职责的映射与限制
```

推荐 ID：阶段 `P01`、任务 `P01-T001`、建议/决定 `D001`、运行 `<日期>-<主题>-<序号>`。

## 核心规则（摘要）

- **权威来源唯一。** roadmap 管目标，phase plan 管任务状态与验收，state 管当前工作位置，decisions 管建议与决定，run 管执行事实，archive index 管保存与备份状态。
- **恢复依据来自文件与证据。** 摘要中的推断不得升级为已验证事实。
- **按子任务判断进度。** 一轮运行成功 ≠ 任务完成；一个任务完成 ≠ 阶段完成。
- **新建议有状态。** 先登记证据与候选建议，接受后才更新受影响的计划。
- **运行、封存、提交、备份分别陈述。** 已提交可能尚未备份，不得互相冒充；核验按用户请求进行，未核验的如实写明。
- **运行来源是历史身份。** 记录执行源码基准 commit 加实际差异，不要求等于当前 HEAD。
- **封存证据不可变。** 后续分析、解释修订和协议变更使用新记录并链接旧证据。
- **授权边界沿用。** 使用本 skill 本身不授权租机、启动实验、迁移文件、commit 或 push。
- **按请求停止。** 每个请求有明确完成点；到达后结束本轮，不追加未要求的检查、打包、Git 操作或核验。未知事项可保持未知。

## 安装与使用

`research-maintainer/` 是一个符合 Agent Skills 约定的 skill 目录（`SKILL.md` + `name`/`description` frontmatter + 按需加载的 `references/`），把它整体放入所使用的 agent 的 skills 目录即可，例如 `~/.codex/skills/` 或 `~/.claude/skills/`。`agents/openai.yaml` 为 Codex 提供显示名与默认 prompt。

使用示例：

```text
用 $research-maintainer 接手这个科研项目，恢复当前进度并维护计划、决策与实验来源。
```

典型调用场景：

- **初始化 / 规范化**：把已有项目的散乱记录映射到标准结构。未获迁移授权时只给出具体映射与差异，不移动文件。
- **接手 / 继续**：先读 `RESEARCH.md`、`research/state.md`、当前 phase plan 和相关未决建议，再核对实际文件、Git 状态与运行证据，输出简短恢复报告。
- **执行工作**：为本次工作关联 task ID（必要时创建 run ID），实际开始后才标 `in_progress`，更新恢复位置与下一动作。
- **暂停 / 交接 / 收尾**：核对真实产物与进程，更新任务状态、证据链接、未解决项与精确恢复点；未完成的核验记录原因，不补写为成功。记录齐备即结束本轮，不追加后续检查。

状态取值：阶段 `planned / active / completed / cancelled`，任务 `todo / in_progress / blocked / paused / done / cancelled`，建议 `proposed / accepted / rejected / superseded`，运行 `prepared / running / succeeded / failed / interrupted / unknown`。

## 不做什么

- 不授权实验、租机、文件迁移、commit 或 push —— 这些取决于用户本次请求与既有授权；
- 不默认执行封存、打包、备份或恢复演练 —— 只按用户当次请求的范围执行；
- 不替代代码实现、独立验证流程或项目的开发规范；
- 不修改已封存包及其内部结果，也不因目录规范自动上传数据；
- 项目数据集、模型和历史文件名不决定 skill 的规则，科学协议内容由项目自身确定。

## 核验情景

`SKILL.md` 末尾列出可用于回归核对的具体情景，例如：阶段只完成两个子任务时，新 agent 应能定位第三个子任务，既不从头重跑也不宣称阶段完成；封存后完成 push 时，备份状态在外部索引更新，封存摘要不被重写；用户要求结束工作时，本轮不新增检查项，未知的保持未知，且不把取消项写成待办。
