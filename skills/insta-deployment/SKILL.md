---
name: insta-deployment
description: 根据项目代码根目录下的 acs-config.json 读取 Deployment 信息，确认 Deployment 存在后，将当前镜像 tag 末尾的 commit id 替换为当前 Git 最新提交的前六位，并使用 kubectl 更新 Deployment 镜像。
---

# insta-deployment

## Purpose

当用户希望把当前项目对应的 Kubernetes Deployment 更新到“当前 Git 分支最新提交对应的镜像 tag”时，使用这个 skill。

这个 skill 只负责解释执行流程、识别关键变量、生成应执行的 kubectl 命令，不直接在 skill 内定义脚本。

## Inputs

从项目代码根目录（或当前工作目录）下的 `acs-config.json` 读取：

- `DEPLOYMENT_NAME`：目标 Deployment 名称，必填
- `NAMESPACE`：目标命名空间，选填，默认 `default`
- `CONTAINER_NAME`：目标容器名，选填

## Default Rules

- 如果没有 `NAMESPACE`，默认使用 `default`
- 如果没有 `CONTAINER_NAME`，优先选择与 `DEPLOYMENT_NAME` 同名的容器
- 如果没有同名容器，则选择 Deployment 中第一个容器
- 不直接修改 Pod，只更新 Deployment
- 只替换镜像 tag 中最后一个 `-` 后面的 commit id 片段
- 保留镜像仓库地址和 tag 前缀不变

## Workflow

1. 操作前先更新 Git 仓库，拉取最新代码保持最新（如执行 `git pull`）。
2. 读取项目代码根目录（或当前工作目录）下的 `acs-config.json`。
3. 从 JSON 中获取 `DEPLOYMENT_NAME`。
4. 从 JSON 中获取 `NAMESPACE`，若不存在则使用 `default`。
5. 检查目标 Deployment 是否存在。
6. 读取 Deployment 中的容器列表。
7. 确定要更新的容器名：
   - 优先使用 `acs-config.json` 中的 `CONTAINER_NAME`
   - 否则优先使用与 Deployment 同名的容器
   - 否则使用第一个容器
8. 读取当前容器镜像。
9. 获取当前 Git 仓库最近一次提交的 short commit id，并取前六位。
10. 如果 `acs-config.json` 中存在 `CONTAINER_NAME`，则检查新镜像是否已在镜像仓库中构建完成（参见"检查镜像是否已构建"步骤）。如果镜像尚未构建完成，停止执行并提示用户。
11. 询问用户要发布的镜像版本（参见"确认发布版本"步骤）。
12. 解析当前镜像 tag，根据用户选择的版本生成新镜像名。
13. 执行 `kubectl set image` 更新 Deployment。
14. 执行 `kubectl rollout status` 确认发布完成。
15. 输出 Deployment、namespace、container、旧镜像、新镜像。

## Image Tag Replacement Rule

示例：

- 当前镜像：`repo/app:dev-d5bd4f`
- 最新 commit：`7e6122`
- 新镜像：`repo/app:dev-7e6122`

再例如：

- 当前镜像：`repo/app:mcp-main-7dde47`
- 最新 commit：`7e6122`
- 新镜像：`repo/app:mcp-main-7e6122`

规则是：

- 保留 `:` 前面的镜像仓库部分
- 保留 tag 中最后一个 `-` 之前的全部前缀
- 只替换最后一个 `-` 之后的 commit 片段

## Kubernetes Commands

### 1. 检查 Deployment 是否存在

```bash
kubectl get deployment ${DEPLOYMENT_NAME} -n ${NAMESPACE}
```

### 2. 查看 Deployment 详情

```bash
kubectl describe deployment ${DEPLOYMENT_NAME} -n ${NAMESPACE}
```

### 3. 查看 Deployment 中容器名称

```bash
kubectl get deployment ${DEPLOYMENT_NAME} -n ${NAMESPACE} -o jsonpath='{.spec.template.spec.containers[*].name}'
```

### 4. 查看当前镜像

如果已知容器名：

```bash
kubectl get deployment ${DEPLOYMENT_NAME} -n ${NAMESPACE} -o jsonpath="{.spec.template.spec.containers[?(@.name=='${CONTAINER_NAME}')].image}"
```

如果只想快速看所有容器镜像：

```bash
kubectl get deployment ${DEPLOYMENT_NAME} -n ${NAMESPACE} -o jsonpath='{.spec.template.spec.containers[*].image}'
```

### 5. 检查镜像是否已构建

如果 `acs-config.json` 中存在 `CONTAINER_NAME`，在执行镜像替换前，使用 `oras` 查询镜像仓库中的 tag 列表，确认新 tag 已存在：

1. 先根据步骤 9 获取最新的 6 位 commit id。
2. 根据步骤 11 的规则，构造出新的镜像 tag（即替换最后一段 commit id 后的完整 tag）。
3. 执行以下命令获取仓库中所有已构建的 tag：

```bash
oras repo tags insta-registry.cn-shanghai.cr.aliyuncs.com/dev/${CONTAINER_NAME}
```

4. 检查输出中是否包含新构造的 tag。如果不存在，说明镜像尚未构建完成，应停止执行。

### 6. 确认发布版本

在执行镜像更新前，使用 `AskUserQuestion` 询问用户要发布的镜像 tag：

1. 先根据当前镜像 tag 的规则，计算出默认建议 tag（即用最新 commit id 替换最后一段）。
2. 同时列出 `oras repo tags` 返回的最近几个 tag 作为可选参考。
3. 向用户提问：

   - 问题：`要发布的镜像版本是？`
   - 选项一：`使用默认（<默认tag>）`（推荐） — 使用 commit id 替换后的 tag
   - 选项二：`自定义 tag` — 用户手动输入完整的镜像 tag
   - 如果用户选择默认，按原规则生成新镜像名。
   - 如果用户选择自定义，使用用户输入的 tag 替换当前镜像的整个 tag 部分（`:` 之后的内容），生成新镜像名。

### 7. 更新镜像

```bash
kubectl set image deployment/${DEPLOYMENT_NAME} ${CONTAINER_NAME}=${NEW_IMAGE} -n ${NAMESPACE}
```

### 8. 查看 rollout 状态

```bash
kubectl rollout status deployment/${DEPLOYMENT_NAME} -n ${NAMESPACE}
```

### 9. 查看 rollout 历史

```bash
kubectl rollout history deployment/${DEPLOYMENT_NAME} -n ${NAMESPACE}
```

### 10. 回滚 Deployment

```bash
kubectl rollout undo deployment/${DEPLOYMENT_NAME} -n ${NAMESPACE}
```

## Failure Conditions

遇到以下情况时应停止执行并明确报错：

- `acs-config.json` 不存在或不是合法的 JSON 格式
- `DEPLOYMENT_NAME` 缺失或为空
- 当前目录不是 Git 仓库
- 无法获取最新 commit id
- Deployment 不存在
- Deployment 中没有容器
- 当前镜像为空
- 当前镜像没有 tag
- tag 不包含 `-`，无法替换最后一段 commit id
- 指定的 `CONTAINER_NAME` 不存在
- 镜像仓库中不存在新构造的 tag，说明镜像尚未构建完成

## Output Format

执行时应明确展示：

- Deployment: `<name>`
- Namespace: `<namespace>`
- Container: `<container>`
- Current image: `<old-image>`
- New image: `<new-image>`

更新后应继续展示：

- `kubectl rollout status` 的结果
- 如果失败，提示可使用回滚命令

## Notes

- Pod 通常由 ReplicaSet / Deployment 控制，直接修改 Pod 不是推荐做法。
- 需要更新镜像时，应始终更新 Deployment 模板。
- 如果 Deployment 是多容器的，必须明确容器名，避免误更新。
- 如果镜像 tag 规范不一致，应先确认 tag 命名规则，再决定是否执行替换。
- 更新最新镜像指的是更换镜像tag，不需要考虑其他内容的更新，除非用户指定切换仓库或者镜像名称。
