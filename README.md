# insta-develop-skills

Insta 研发团队 Claude Code 技能集合。本仓库沉淀了团队在部署、排障、任务规划等场景下高频使用的 skill，方便在各个项目中复用。

## 安装指南

在任意项目目录下，通过 [`skills`](https://github.com/anthropics/skills) CLI 一键安装本仓库的技能：

```bash
npx skills add Instaon/insta-develop-skills
```

执行后技能会被安装到当前项目对应的 `.claude/skills` 目录，安装完成即可在 Claude Code 中使用。

## 包含的技能

| 技能 | 目录 | 说明 |
| --- | --- | --- |
| **insta-deployment** | [`skills/insta-deployment`](skills/insta-deployment) | 管理项目对应的 Kubernetes Deployment 镜像：更新到最新 Git 提交对应的版本、部署指定镜像版本、查询当前部署版本、回滚。配置从项目根目录的 `acs-config.json` 读取。 |
| **insta-error-triage** | [`skills/insta-error-triage`](skills/insta-error-triage) | 分析错误日志并分级处理：结合项目代码定位根因，必要时用 kubectl 拉取对应 Deployment 的运行日志补充上下文；紧急问题通过 GitHub MCP 提交 issue 并 @Claude01，非紧急问题输出结构化分析报告。 |
| **insta-api-smoketest** | [`skills/insta-api-smoketest`](skills/insta-api-smoketest) | 基于 `insta-backendApi-docs` 接口文档，用 curl 对后端接口做轻量黑盒冒烟测试（默认增量，全量需明确要求）。 |
| **insta-test-port** | [`skills/insta-test-port`](skills/insta-test-port) | 从 `insta-backendApi-docs/test-port-registry.yaml` 分配下一个可用测试端口：拉最新代码、max+1、回写登记并提交（推送失败可忽略），回报端口与推送结果。 |
| **planning-with-files-zh** | [`.agents/skills/planning-with-files-zh`](.agents/skills/planning-with-files-zh) | 基于 Manus 风格的文件规划系统，用于组织和跟踪复杂任务的进度（task_plan.md / findings.md / progress.md），支持 `/clear` 后的会话自动恢复。 |

## 目录结构

```
.
├── skills/                      # 项目级技能
│   ├── insta-deployment/
│   ├── insta-error-triage/
│   ├── insta-api-smoketest/
│   └── insta-test-port/
└── .agents/skills/              # Agent 级技能
    └── planning-with-files-zh/
```

每个技能目录下均包含一份 `SKILL.md`，记录了触发条件与执行流程。
