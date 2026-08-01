---
title: "Git PR 使用 Squash and Merge 前，什么时候应该 Rebase？"
date: 2026-08-01T18:04:00+08:00
draft: false
tags: ["Git", "GitHub", "代码评审", "开发工作流"]
categories: ["技术文章"]
---

## 先说结论

在 GitHub 中使用 **Squash and Merge** 合并 PR，并不意味着每个 PR 都必须先执行 `rebase`。

两者解决的问题不同：

- `rebase`：把当前分支重新接到目标分支最新提交之后，必要时顺便整理提交顺序。
- `Squash and Merge`：合并 PR 时，把 PR 中的多个提交压缩成目标分支上的一个提交。

所以，是否需要 rebase，主要看分支是否落后、是否出现冲突、是否需要在合并前验证最新代码，而不是看 PR 里有多少个 commit。

## 一个常见的误区

很多人看到 PR 里有这样的提交记录：

```text
feat: add login page
fix: adjust login style
fix: handle empty password
WIP
```

于是认为合并前必须执行：

```bash
git rebase -i HEAD~4
```

其实，如果仓库最终使用 **Squash and Merge**，GitHub 会在合并时把这些提交压成一个提交。上面的提交历史虽然不够整齐，但通常不会污染 `develop` 或 `main` 分支。

为了整理提交信息而 rebase，属于可选操作；为了让 PR 基于最新目标分支并通过测试，才是更常见、更有实际价值的 rebase 场景。

## 什么时候应该在合并前 Rebase？

### 1. 目标分支已经有较多新提交

假设你的 PR 是从 `develop` 创建的：

```text
develop:  A---B---C---D
               \
feature:        E---F
```

其他开发者已经把 `C`、`D` 合并到 `develop`，而你的功能分支还停留在 `B`。这时可以先把功能分支 rebase 到 `origin/develop`：

```bash
git fetch origin
git switch feature/login
git rebase origin/develop
```

结果大致变成：

```text
develop:  A---B---C---D
                       \
feature:                E'--F'
```

这样可以在 PR 合并前，使用目标分支的最新代码运行测试。若有冲突，也能由功能开发者提前处理，而不是把冲突留给合并者。

### 2. 目标分支的改动会影响你的代码

如果 `develop` 最近修改了接口、数据库结构、公共组件或配置，而你的 PR 正好依赖这些内容，建议 rebase 后重新运行测试。

这里的重点不是“提交图看起来更直”，而是确认：

```text
最新 develop + 当前功能代码
```

能否一起正常编译、测试和运行。

### 3. 仓库要求分支必须保持最新

有些仓库启用了分支保护规则，例如：

- PR 必须基于目标分支最新提交；
- 必须通过最新代码上的 CI；
- 禁止通过带冲突的分支合并。

如果 GitHub 提示分支落后，或者出现 `This branch is out-of-date`，可以选择在 GitHub 上更新分支，也可以在本地执行 rebase。具体以团队约定为准。

### 4. 需要在合并前整理提交内容

虽然 Squash and Merge 会处理最终历史，但在某些情况下，PR 中的提交本身也需要便于评审。例如：

- 提交中包含明显无关的调试代码；
- 提交顺序导致代码无法逐步构建；
- 评审者需要按提交理解一组较大的改动；
- 团队要求每个 commit 都能独立通过检查。

这时可以使用交互式 rebase：

```bash
git rebase -i origin/develop
```

然后根据需要使用 `pick`、`reword`、`edit`、`squash` 或 `fixup` 整理提交。

不过，普通业务 PR 通常不必为了“看起来专业”而过度整理。代码是否正确、改动是否容易评审，比提交数量更重要。

## 什么时候不必 Rebase？

以下情况通常可以直接使用 Squash and Merge：

- 功能分支刚从目标分支创建出来，目标分支没有明显变化；
- CI 已经基于最新目标分支运行并通过；
- PR 没有冲突，且仓库允许直接合并；
- 分支只是个人使用，提交历史不需要单独保留；
- rebase 只是为了把两个普通提交变成一个，而 GitHub 已经会 Squash。

尤其要注意：**Squash and Merge 不要求 PR 必须只有一个 commit。** PR 可以有多个开发过程中的提交，最终由 GitHub 生成一个合并提交。

## 推荐的实际操作流程

下面以 `feature/login` 合并到 `develop` 为例：

### 1. 获取远程最新信息

```bash
git fetch origin
```

`fetch` 只更新本地的远程跟踪分支，不会修改当前工作区，适合先安全地查看远程状态。

### 2. 切换到功能分支并确认工作区干净

```bash
git switch feature/login
git status
```

如果还有未提交的改动，先提交，或者使用 `git stash` 临时保存。不要在一堆未确认的本地改动上直接 rebase。

### 3. Rebase 到目标分支

```bash
git rebase origin/develop
```

没有冲突时，Git 会自动完成。完成后运行项目的格式检查、单元测试和构建命令。

### 4. 处理冲突

如果出现冲突，先查看状态：

```bash
git status
```

编辑冲突文件，删除 `<<<<<<<`、`=======`、`>>>>>>>` 等标记，确认保留的代码正确，然后继续：

```bash
git add path/to/conflicted-file
git rebase --continue
```

如果发现这次 rebase 不应该进行，随时可以取消：

```bash
git rebase --abort
```

取消后，功能分支会回到 rebase 开始前的状态。

### 5. 推送更新后的分支

rebase 会重写功能分支上的 commit ID，所以普通 `git push` 可能会被拒绝。确认本地测试通过后，使用：

```bash
git push --force-with-lease origin feature/login
```

优先使用 `--force-with-lease`，不要习惯性使用 `--force`。前者会检查远程分支是否出现了你不知道的新提交，能避免误覆盖他人的工作。

### 6. 在 GitHub 上使用 Squash and Merge

PR 更新后，重新等待 CI 通过，再点击 **Squash and merge**。合并完成后，目标分支通常只会增加一个提交，提交信息可以在合并前编辑为清晰的功能描述。

## Rebase 时最需要注意什么？

### 不要随意 Rebase 已被多人共同使用的分支

Rebase 会改写提交历史。如果 `feature/login` 已经被其他人拉取并继续开发，直接 rebase 并强制推送，会让他们本地的分支与远程历史分叉。

个人功能分支通常问题不大；共享分支、公共发布分支和已经被其他人依赖的分支，应先沟通，或者改用普通 merge。

### Rebase 后要重新检查测试结果

即使 rebase 没有冲突，也不代表代码一定正确。Git 只知道文本是否能应用，不知道业务逻辑是否仍然成立。特别是接口和数据库结构发生变化时，必须重新运行相关测试。

### 不要把 `git pull --rebase` 当成万能命令

`git pull --rebase` 适合在个人分支上同步远程变更，但在复杂 PR 中，显式执行下面的步骤更容易看清发生了什么：

```bash
git fetch origin
git rebase origin/develop
```

## 一个简单的判断方法

提交 PR 前可以问自己三个问题：

1. 目标分支在我开始开发后是否有影响当前功能的改动？
2. PR 是否存在冲突，或者仓库是否要求分支保持最新？
3. 我是否需要在最新目标分支上重新验证代码？

只要其中一个问题的答案是“是”，就值得考虑 rebase。三个问题都是“否”时，通常可以直接让 GitHub 执行 Squash and Merge。

## 总结

`rebase` 和 `Squash and Merge` 经常一起出现，但它们不是一套必须绑定执行的命令：

- 想同步目标分支、提前处理冲突、在最新代码上测试：使用 `rebase`；
- 想让目标分支保留一个清晰的 PR 提交：使用 `Squash and Merge`；
- 只是因为 PR 有多个 commit：通常不需要 rebase；
- rebase 个人分支后要强制推送：使用 `--force-with-lease`，并确认没有覆盖他人的提交。

一个实用的默认流程是：**开发 → 提交 PR → 目标分支有变化时 rebase → 解决冲突并测试 → 推送 → Squash and Merge**。把 rebase 当成“同步和验证工具”，而不是提交数量不够整齐时的仪式动作，工作流会简单很多。
