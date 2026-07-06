---
name: insta-deployment
description: 管理项目对应的 Kubernetes Deployment 镜像：更新到最新 Git 提交对应的版本、部署用户指定的镜像版本、查询当前部署版本、回滚。先定位目标项目（用户指定的项目名，或当前所在项目），再从项目根目录的 acs-config.json 读取配置。当用户提到"更新部署 / 发一下最新代码 / 部署某个版本 / 发布某个 tag / 回滚 / 看看现在部署的是什么版本"等意图时使用本 skill。
---

# insta-deployment

## Purpose

管理当前项目对应的 Kubernetes Deployment 的镜像版本。本 skill 定义了四种操作及其执行流程，只生成并执行 kubectl / oras 命令，不在 skill 内定义脚本。

## 核心约束：只更换版本号，不动镜像地址

镜像的完整结构为：

```
<镜像地址>:<tag 前缀>-<版本号>
例如：insta-registry.cn-shanghai.cr.aliyuncs.com/dev/app:mcp-main-7dde47
      └────────────── 镜像地址 ──────────────────┘ └ 前缀 ┘ └版本号┘
```

- **镜像地址**（`:` 之前的全部，包含仓库域名、命名空间、镜像名称）：**任何操作都不修改**，除非用户明确要求切换仓库或镜像名称。
- **tag 前缀**（tag 中最后一个 `-` 之前的部分，如 `dev`、`mcp-main`）：更新和部署操作均保持不变。
- **版本号**（tag 中最后一个 `-` 之后的 commit id 片段）：这是唯一被替换的部分。

之所以有这条约束，是因为镜像地址和 tag 前缀由 CI 体系决定，改动它们会指向完全不同的构建产物；本 skill 的职责仅仅是把同一条构建流水线产出的镜像切换到另一个版本。

## 意图路由

先判断用户意图，选择对应流程，不要混用：

| 用户意图示例 | 执行流程 |
|---|---|
| "更新到最新代码"、"发一下最新提交"、"更新部署" | 流程 A：更新到最新提交 |
| "部署 xxx 这个版本"、"发布 tag 为 yyy 的镜像" | 流程 B：部署指定版本 |
| "现在部署的是什么版本"、"看看线上镜像" | 流程 C：查询当前版本 |
| "回滚"、"退回上一个版本" | 流程 D：回滚 |

意图不明确时，先用 AskUserQuestion 确认，再进入流程。

## 流程 A：更新到最新提交

把 Deployment 的镜像版本号替换为目标项目 Git 分支最新提交的前六位。

1. 执行《定位目标项目》。
2. 在目标项目目录下执行 `git pull` 拉取最新代码。
3. 执行《读取配置》。
4. 执行《定位 Deployment 与容器》。
5. 执行《读取当前镜像》，按"核心约束"一节拆解出镜像地址、tag 前缀、版本号。
6. 在目标项目目录下获取最新提交的 short commit id，取前六位：`git rev-parse --short=6 HEAD`。
7. 用该六位 commit id 替换版本号，生成新镜像（镜像地址与 tag 前缀原样保留）。
8. 执行《校验镜像已构建》，确认新 tag 已存在于镜像仓库；不存在则停止并提示用户镜像尚未构建完成。
9. 执行《确认发布》。
10. 执行《更新镜像并观察发布》。

版本号替换示例：

- 当前镜像 `repo/app:dev-d5bd4f`，最新提交 `7e6122` → 新镜像 `repo/app:dev-7e6122`
- 当前镜像 `repo/app:mcp-main-7dde47`，最新提交 `7e6122` → 新镜像 `repo/app:mcp-main-7e6122`

## 流程 B：部署指定版本

用户已明确指定要发布的版本，不需要 git pull，也不需要计算 commit id。

1. 执行《定位目标项目》。
2. 执行《读取配置》。
3. 执行《定位 Deployment 与容器》。
4. 执行《读取当前镜像》。
5. 解析用户给出的版本：
   - 如果用户给的是**版本号**（如 `7e6122`），只替换 tag 中最后一段版本号。
   - 如果用户给的是**完整 tag**（如 `mcp-main-7e6122`），替换 `:` 之后的整个 tag。
   - 如果用户给的内容包含镜像地址且与当前地址不同，视为切换仓库/镜像名称——这超出常规操作，必须先向用户确认后才能继续。
6. 执行《校验镜像已构建》，确认目标 tag 存在；不存在则停止并提示。
7. 执行《确认发布》。
8. 执行《更新镜像并观察发布》。

## 流程 C：查询当前版本

只读操作，不修改任何资源。

1. 执行《定位目标项目》。
2. 执行《读取配置》。
3. 执行《定位 Deployment 与容器》。
4. 执行《读取当前镜像》，输出 Deployment、namespace、container、当前镜像，并指出 tag 中的版本号对应的 commit。

## 流程 D：回滚

1. 执行《定位目标项目》。
2. 执行《读取配置》。
3. 查看发布历史：

   ```bash
   kubectl rollout history deployment/${DEPLOYMENT_NAME} -n ${NAMESPACE}
   ```

4. 向用户展示历史版本并确认回滚目标。
5. 执行回滚：

   ```bash
   kubectl rollout undo deployment/${DEPLOYMENT_NAME} -n ${NAMESPACE}
   ```

   如需回到指定版本，追加 `--to-revision=<N>`。
6. 执行 `kubectl rollout status` 观察结果。

## 共享步骤库

以下步骤被各流程按名称引用。

### 定位目标项目

先确定本次操作针对哪个项目，再进入后续步骤：

- 用户指定了项目时，以用户指定的项目为准，找到该项目的代码目录。
- 用户未指定时，使用当前所在的项目。
- 无法确定目标项目时，向用户确认，不要猜。

后续所有步骤（读取配置、git 操作等）都在目标项目的根目录下执行。

### 读取配置

读取目标项目根目录下的 `acs-config.json`：

- `DEPLOYMENT_NAME`：目标 Deployment 名称，必填。缺失或文件不是合法 JSON 时停止并报错。
- `NAMESPACE`：目标命名空间，选填，默认 `default`。
- `CONTAINER_NAME`：目标容器名，选填。

### 定位 Deployment 与容器

1. 确认 Deployment 存在，不存在则停止并报错：

   ```bash
   kubectl get deployment ${DEPLOYMENT_NAME} -n ${NAMESPACE}
   ```

2. 读取容器列表：

   ```bash
   kubectl get deployment ${DEPLOYMENT_NAME} -n ${NAMESPACE} -o jsonpath='{.spec.template.spec.containers[*].name}'
   ```

3. 确定目标容器，按优先级：
   1. `acs-config.json` 中的 `CONTAINER_NAME`（若指定的容器不存在于列表中，停止并报错）；
   2. 与 `DEPLOYMENT_NAME` 同名的容器；
   3. 列表中的第一个容器。

   多容器 Deployment 必须明确容器名后再操作，避免误更新。

### 读取当前镜像

```bash
kubectl get deployment ${DEPLOYMENT_NAME} -n ${NAMESPACE} -o jsonpath="{.spec.template.spec.containers[?(@.name=='${CONTAINER_NAME}')].image}"
```

镜像为空、没有 tag、或 tag 不含 `-` 无法拆出版本号时，停止并向用户说明当前 tag 的实际格式，确认命名规则后再继续。

### 校验镜像已构建

无论新 tag 来自 commit id 计算（流程 A）还是用户指定（流程 B），发布前都要确认它已在镜像仓库中构建完成：

```bash
oras repo tags ${IMAGE_REPO}
```

其中 `${IMAGE_REPO}` 是当前镜像 `:` 之前的镜像地址部分。输出中不包含目标 tag 时，停止执行并提示用户镜像尚未构建完成。

### 确认发布

发布是影响线上环境的操作，执行前必须用 AskUserQuestion 确认：

- 问题：`要发布的镜像版本是？`
- 选项一：`使用 <新tag>`（推荐）—— 流程计算或用户指定的目标 tag
- 选项二：`自定义 tag` —— 用户手动输入完整 tag；输入后同样需要通过《校验镜像已构建》
- 提问时同时列出 `oras repo tags` 返回的最近几个 tag 作为参考。

### 更新镜像并观察发布

只更新 Deployment 模板，不直接修改 Pod（Pod 由 ReplicaSet 控制，直接改会被覆盖）：

```bash
kubectl set image deployment/${DEPLOYMENT_NAME} ${CONTAINER_NAME}=${NEW_IMAGE} -n ${NAMESPACE}
kubectl rollout status deployment/${DEPLOYMENT_NAME} -n ${NAMESPACE}
```

rollout 失败时，提示用户可用流程 D 回滚。

## Failure Conditions

任一流程中遇到以下情况，停止执行并明确报错：

- `acs-config.json` 不存在、不是合法 JSON、或 `DEPLOYMENT_NAME` 缺失
- Deployment 不存在、无容器、或指定的 `CONTAINER_NAME` 不存在
- 当前镜像为空或没有 tag
- 镜像仓库中不存在目标 tag（镜像尚未构建完成）

仅流程 A 额外要求：

- 目标项目必须是 Git 仓库且能获取最新 commit id
- 当前 tag 必须包含 `-`，否则无法定位版本号段

## Output Format

执行更新/部署后，明确展示：

- Deployment: `<name>`
- Namespace: `<namespace>`
- Container: `<container>`
- Current image: `<old-image>`
- New image: `<new-image>`
- `kubectl rollout status` 的结果；失败时附回滚提示
