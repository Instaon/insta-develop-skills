---
name: project-knowledge-base
description: 维护并同步项目知识库文档，通过 Git 子模块管理 insta-backendApi-docs 并组织项目文档，提供项目地图查询。
---

# 维护项目知识库技能 (project-knowledge-base)

此技能用于在项目中规范地初始化、同步、编辑和提交项目文档到统一的知识库仓库 `insta-backendApi-docs`（以 Git 子模块的形式嵌入）。

## 核心配置与原则

1. **子模块地址**：
   - HTTPS 协议：`https://github.com/Instaon/insta-backendApi-docs.git`
   - SSH 协议：`git@github.com:Instaon/insta-backendApi-docs.git`
2. **位置约定**：
   - 子模块将被拉取至主项目根目录下的 `insta-backendApi-docs` 文件夹中。
   - 当前项目的文档应维护在 `insta-backendApi-docs/<project-name>/` 目录下。项目名称通常是当前主项目 Git 仓库根目录的名称（可以通过 `basename $(git rev-parse --show-toplevel)` 获得，如 `insta-develop-skills`）。
   - 项目关系地图文件位于 `insta-backendApi-docs/project-map.md`。

---

## 场景操作指引 (遇到什么情况，做什么，怎么做)

### 场景一：首次在项目中使用知识库，或发现知识库子模块未就绪
* **要做什么**：初始化并配置 Git 子模块，确保文档仓库拉取成功并创建本项目文档目录。
* **怎么做**：
  1. 检查主项目根目录下是否存在 `insta-backendApi-docs` 目录，或者 `.gitmodules` 文件中是否有相应配置。
  2. 若尚未配置，在主项目根目录下执行 Git 命令尝试添加子模块（优先使用 HTTPS 地址）：
     ```bash
     git submodule add https://github.com/Instaon/insta-backendApi-docs.git
     ```
  3. 如果在添加过程中，由于权限或凭证报错，则回退尝试使用 SSH 地址重新添加：
     ```bash
     git submodule add git@github.com:Instaon/insta-backendApi-docs.git
     ```
  4. 执行初始化和更新，确保子模块内容被拉取到本地：
     ```bash
     git submodule update --init --recursive
     ```
  5. 检查子模块内是否存在以当前项目命名的文件夹（即 `insta-backendApi-docs/<project-name>/`）。若不存在，则在子模块内创建该文件夹，并添加一个初始的 `README.md`，以此来声明当前项目的文档空间：
     ```bash
     mkdir -p insta-backendApi-docs/<project-name>
     echo "# <project-name> 文档库" > insta-backendApi-docs/<project-name>/README.md
     ```

### 场景二：在开始任何文档新增、修改或编辑之前
* **要做什么**：前置同步，拉取最新文档，避免多人协作时产生冲突。
* **怎么做**：
  1. 必须优先保持知识库子模块是最新状态。进入子模块目录：
     ```bash
     cd insta-backendApi-docs
     ```
  2. 确保子模块未处于游离指针 (detached HEAD) 状态，切换并拉取对应的跟踪分支（默认通常是 `main` 或者是 `master`）：
     ```bash
     git checkout main
     ```
     *(注：如果远端默认分支是 master，则执行 `git checkout master`)*
  3. 从远端拉取最新更改并合并：
     ```bash
     git pull origin main
     ```
  4. 确认没有冲突后，回到主项目或开始在该文件夹下编辑文档。

### 场景三：需要理清当前项目与其他项目之间的依赖或调用关系
* **要做什么**：查询并阅读项目地图。
* **怎么做**：
  1. 直接打开并查阅文件 `insta-backendApi-docs/project-map.md`。
  2. 通过此地图文件分析当前项目（例如 `insta-develop-skills`）在整个生态或微服务架构中的位置与关联，以此为基准编写或校对文档。

### 场景四：准备提交主仓库代码，并且子模块中有文档修改
* **要做什么**：更新并推送知识库子模块，并将子模块的指针同步更新至主项目。
* **怎么做**：
  1. 进入子模块目录并检查修改：
     ```bash
     cd insta-backendApi-docs
     git status
     ```
  2. 若确认有文档更新，将变动提交并推送至知识库远端仓库（分支须与拉取分支保持一致，如 `main` 或 `master`）：
     ```bash
     git add .
     git commit -m "docs(<当前项目名>): <您的具体修改说明>"
     git push origin main
     ```
  3. 返回主项目根目录。此时主项目会感知到子模块的 Commit ID 指针发生了变化。
  4. 将子模块的最新指针位置暂存到主项目中，以便随同主项目代码一同提交：
     ```bash
     cd ..
     git add insta-backendApi-docs
     ```
  5. 随后，像往常一样在主项目中进行代码提交（例如 `git commit -m "feat: xxx"` 并 `git push`），使得主项目保存的子模块指针同步为最新的 Commit。
