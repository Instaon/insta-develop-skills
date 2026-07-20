# insta-develop-skills 项目说明

> 本文档回答「这个项目是做什么的」，面向新成员与需要复用技能的同学。

## 一句话定位

`insta-develop-skills` 是 **Insta 研发团队的 Claude Code 技能（Skills）集合仓库**，把团队在部署发布、线上排障、复杂任务规划等高频研发场景下沉淀的最佳实践，封装成可被 Claude Code 直接调用的 Skill，方便在各个项目里统一复用。

## 解决什么问题

研发流程中有很多"标准动作"：发版时怎么换 K8s 镜像版本、线上报错怎么定位分级、复杂任务怎么不丢上下文地推进。这些动作如果只靠口口相传或每次手动操作，容易出现：

- 不同人操作不一致，容易误改镜像地址、误删资源；
- 排障经验分散在个人脑子里，根因分析质量参差；
- 多步骤任务在长会话或 `/clear` 后丢失上下文。

本仓库把这些动作固化成技能：技能本身不写脚本，而是定义**触发条件 + 执行流程 + 安全约束**，由 Claude Code 在真实项目里按流程执行（生成并运行 kubectl / git 等命令）。这样团队所有人触发的是同一套经过审定的流程。

## 包含哪些技能

| 技能 | 位置 | 类别 |
| --- | --- | --- |
| insta-deployment | `skills/insta-deployment/` | 项目级（随仓库分发） |
| insta-error-triage | `skills/insta-error-triage/` | 项目级 |
| insta-api-smoketest | `skills/insta-api-smoketest/` | 项目级 |
| insta-test-port | `skills/insta-test-port/` | 项目级 |
| planning-with-files-zh | `.agents/skills/planning-with-files-zh/` | Agent 级 |

### 1. insta-deployment —— Kubernetes 镜像版本管理

管理当前项目对应 Deployment 的镜像版本，定义了四种操作，**只生成并执行 kubectl / oras 命令，不在技能内置脚本**。

核心安全约束：**只换版本号，不动镜像地址**。镜像形如 `<镜像地址>:<tag前缀>-<版本号>`（如 `repo/app:mcp-main-7dde47`），其中镜像地址和 tag 前缀由 CI 体系决定、一律不动，只替换最后一段 commit id 片段，避免指向错误的构建产物。

| 用户意图 | 流程 |
| --- | --- |
| "更新到最新代码 / 发一下最新提交" | A：拉取最新代码 → 取最新 commit 前 6 位 → 替换版本号 → 校验镜像已构建 → 确认 → 发布 |
| "部署 xxx 版本 / 发布 tag yyy" | B：按用户给的版本号或完整 tag 替换 → 校验 → 确认 → 发布 |
| "现在部署的是什么版本" | C：只读查询当前镜像并指出对应 commit |
| "回滚 / 退回上一版" | D：查看 rollout history → 确认 → `kubectl rollout undo` |

配置从目标项目根目录的 `acs-config.json` 读取（`DEPLOYMENT_NAME` 必填，`NAMESPACE` / `CONTAINER_NAME` 选填）。所有发布前都用 `oras repo tags` 校验目标 tag 已存在，并用 AskUserQuestion 二次确认。

### 2. insta-error-triage —— 错误日志分级处置

接收错误日志后完成「分析 → 定位 → 分级 → 处置」全流程，**产出只有两种：紧急问题提交 GitHub issue，非紧急输出结构化分析报告**，不在技能内直接改代码。

- **分析定位**：从日志提取错误类型、频率、触发路径；结合项目代码用堆栈里的文件/函数名检索定位根因；日志不全时从 `acs-config.json` 读配置，用 kubectl 只读拉取 Deployment 日志补充上下文。
- **严重度分级**：只看「是否影响正常功能使用」，不看病面严重程度（ERROR 有兜底可能不算紧急，WARN 也可能在悄悄丢数据）。拿不准时宁可按紧急处理。
- **处置**：
  - 紧急 → 在目标项目仓库创建 issue，正文 `@Claude01` 触发自动化处理，提交前必须先向用户确认内容；
  - 非紧急 → 直接输出报告（错误摘要、根因分析、影响范围、修复复杂度、修复方向）。

### 3. insta-api-smoketest —— 接口文档黑盒冒烟

基于 `insta-backendApi-docs` 的接口文档，用 curl 发 mock 请求做轻量契约验证。默认**增量**（只测近期改动相关接口），用户明确要求时才全量。失败项在「确认实现侧问题 + 影响使用」时才自主建 issue，与 error-triage 策略对齐。

### 4. insta-test-port —— 测试端口分配

从 `insta-backendApi-docs` 根目录的 `test-port-registry.yaml` 领取下一个可用测试端口，避免联调撞端口：

1. 定位可写的 docs 仓库副本并 `pull` 最新
2. 读 `max_used_port`，新端口 = 最大值 + 1
3. 回写 `max_used_port` 与 `allocations`（登记 service / note）
4. **必须本地 commit**；push 尽力而为（网络失败可忽略，不回滚 commit）
5. 向用户回报端口号与推送是否成功

只负责注册表领号，不强制改业务项目配置。

### 5. planning-with-files-zh —— 文件化任务规划系统

借鉴 Manus 的「磁盘工作记忆」理念，用持久化的 Markdown 文件组织复杂任务：

| 文件 | 用途 |
| --- | --- |
| `task_plan.md` | 阶段划分、进度、决策 |
| `findings.md` | 研究发现（**网页/搜索等不可信内容只写这里**） |
| `progress.md` | 会话日志、测试结果 |

通过 UserPromptSubmit / PreToolUse / PostToolUse / Stop 钩子，在每次工具调用前后重新注入计划内容，从而支持 `/clear` 后基于文件自动恢复会话上下文。内置「两步操作规则」「三次失败协议」「读取/写入决策矩阵」等纪律，防止长任务中信息丢失和重复踩坑。

## 技能之间的协作关系

这三个技能并非孤立，而是覆盖了研发闭环的不同环节：

```
planning-with-files-zh   ← 规划与跟踪任意复杂任务的底座
        │
        ├── insta-error-triage   ← 线上报错时定位+分级
        │        │ （紧急）
        │        └── 创建 GitHub issue @Claude01
        │                  │
        │                  ▔▔▔▔▔ ↓ （修复完成）
        │
        └── insta-deployment     ← 修复后的代码，更新/回滚 K8s 镜像版本发布
```

典型链路：排障技能发现紧急问题 → 建 issue 交给 Claude Code 处理修复 → 修复完成后用部署技能把新镜像版本发到 K8s。整个过程都可以用规划技能跟踪进度。

## 安装与使用

在任意项目目录下，通过 [`skills`](https://github.com/anthropics/skills) CLI 一键安装：

```bash
npx skills add Instaon/insta-develop-skills
```

技能会被安装到当前项目的 `.claude/skills`（项目级）或对应 agent 目录（agent 级），安装完成即可在 Claude Code 中通过自然语言触发。

- **部署类操作**需要目标项目根目录有 `acs-config.json`，并能访问对应 K8s 集群与镜像仓库（kubectl / oras）。
- **排障类操作**需要 kubectl 只读权限，以及紧急建 issue 时的 GitHub MCP 权限。
- **规划类操作**纯本地文件，无外部依赖。

## 适用场景与目标用户

- **目标用户**：Insta 研发团队成员，尤其是使用 Claude Code 日常开发、需要发版和排障的同学。
- **适用场景**：
  - 需要标准化发布/回滚 K8s 服务版本；
  - 收到线上错误日志需要快速定位根因并决定是建 issue 还是出报告；
  - 推进多步骤复杂任务，希望跨会话不丢上下文。

## 目录结构

```
.
├── skills/                              # 项目级技能（随仓库分发）
│   ├── insta-deployment/SKILL.md
│   └── insta-error-triage/SKILL.md
├── .agents/skills/                      # Agent 级技能
│   └── planning-with-files-zh/
│       ├── SKILL.md
│       ├── scripts/                     # 会话初始化、完成校验、上下文恢复脚本
│       └── templates/                   # task_plan / findings / progress 模板
└── README.md
```

每个技能目录下的 `SKILL.md` 记录了完整的触发条件、执行流程与失败处理，是技能行为的权威定义。
