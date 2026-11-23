---
title: "Git一文速通"
description: 
date: 2026-11-23T12:30:39+08:00
tags: ["Git命令", "版本控制", "变基"]
categories: ["Git"]
image: 
math: 
license: 
comments: true
draft: false
build:
    list: always    # Change to "never" to hide the page from the list
---

# Git 核心工作流与常用命令指南

## 1. Git 的三大区域

Git 的核心在于文件在不同区域之间的流转。理解这三个区域是掌握 Git 的前提。

| 区域 | 位置 | 解释 |
| ------ |------ |------ |
| **工作区** | 你的本地文件系统 | 你直接编辑、修改代码的地方。 |
| **暂存区** | `.git` 目录下的隐藏文件 | 临时保存待提交的更改（`git add` 后的内容）。 |
| **本地仓库** | `.git` 文件夹 | 存储 Git 提交历史和版本信息（`git commit` 后的内容）。 |

## 2. 获取 Git 仓库

有两种方式获取 Git 仓库：

- **方法一**：使用 `git clone` 从远程服务器克隆。
- **方法二**：在本地目录使用 `git init` 初始化。

## 3. 查看文件状态

Git 中的文件分为两种状态：

- **已跟踪**：已经纳入了版本控制的文件。
- **未跟踪**：没有纳入版本控制的文件。

### git status：查看工作目录下的文件状态

- **Untracked files**：列出未被跟踪的文件，可使用 `git add <file>` 跟踪新文件。
- **Changes to be committed**：列出处于暂存状态的文件，使用 `git commit` 命令之后，这些文件会被提交保存。
- **Changes not staged for commit**：被修改过但是还没有被 `git add` 暂存的改动文件，使用 `git add` 后会暂存最新的修改。

### git status -s：简短查看状态

- `??`：未跟踪文件。
- `A `：新加入跟踪的文件。
- `M `：修改过的已跟踪文件（第一个 M 指示修改已暂存，第二个 M 指示修改未暂存，两个 M 指示该文件在暂存之后有了新的修改）。
- `D `：删除的文件。

## 4. 查看文件修改内容 (git diff)

- `git diff`：对比**工作区**和**暂存区**的区别，也就是**没有 **`git add`** 的内容**是什么。
- `git diff --staged` (或 `--cached`)：对比**已暂存的文件**和**最后一次提交**的内容，也就是**已经 **`git add`** 的内容**是什么。

## 5. 删除文件 (git rm)

### 彻底删除文件

`git rm <file>` 等价于在系统里执行 `rm <file>` 加上 `git add <file>`。

```bash
git rm test.txt
```

### 仅在 Git 仓库中删除但在本地保存

如果误把不该提交的文件（如日志、密码文件）提交了，可以使用此命令将其移出 Git 追踪，但保留本地文件。

```bash
git rm --cached test.txt
```

## 6. 查看提交历史 (git log)

`git log` 默认按时间倒序排列提交，新提交在上边。

- `git log -p`：展示每次提交的具体差异（Diff）。
- `git log --oneline`：将每个提交精简为一行显示。
- `git log --oneline --graph --all --decorate`：**强烈推荐**，在日志旁以 ASCII 图形显示分支与合并历史，非常直观。
- `git log <file>`：查看某个指定文件的提交历史。

## 7. 修改最近一次提交 (git commit --amend)

覆盖最近一次提交，常用于修正提交信息或补充遗漏的文件。

1. **修改提交信息**：
2. **增加提交文件，不修改 commit 信息**：

## 8. 撤销与回退操作

### 将修改从暂存区移除（取消 git add）

- `git restore --staged <file>`：建议使用此新语法。
- `git reset HEAD <file>`：老方法，不建议使用。

### 撤销修改与回退版本

- `git reset HEAD~n`：撤销最近 n 次提交，**保留修改在工作区**。
- `git reset --hard HEAD~n`：撤销最近 n 次提交，**彻底删除修改**，不在工作区保留。
- `git revert <commit>`：用**新增一次提交**的方式撤销某次提交，**安全**，适合已推送到远程的代码。
    - 对于 merge 提交，需要使用 `-m` 指定父提交基线：`git revert -m N <merge-commit>`
        - `N = 1`：回滚到执行 merge 时，当前分支所在的 HEAD，**几乎都是用这个**。
        - `N = 2`：回滚到被合并分支的 HEAD，几乎不用这个。


⚠️ **核心区别**：

- `reset` 会粗暴地将指针指回历史提交，容易造成代码丢失和分支冲突，建议只有明确想要删除还没有推送的本地提交时使用。
- `revert` 相当于帮你把提交过的代码执行了一次反向操作，之后再进行一次新提交，这种方式安全，而且可以通过对当前 revert 再进行一次 revert 来取消这次回滚。


## 9. 远程仓库操作

### 管理远程仓库

- 查看远程仓库：`git remote -v`
- 添加远程仓库：`git remote add <shortname> <url>`
- 重命名远程仓库：`git remote rename <old_name> <new_name>`
- 删除远程仓库：`git remote remove <remote>`

### 查看远程仓库详细信息

`git remote show <remote>`：展示远程仓库的信息，以及分支的跟踪情况。

| 分支状态 | 含义 |
| ------ |------ |
| tracked | 本地已同步 |
| new | 远程新建，本地未知 |
| stale | 远程已删，本地残留 |

### 获取与推送代码

- `git fetch <remote>`：获取远程仓库中所有分支的引用，但是**并不会执行合并**。
- `git push <remote> <branch>`：将本地分支推送到远程仓库。

## 10. 标签管理 (git tag)

### 查看标签

- `git tag`：列出所有的标签。
- `git tag -l "v1.8.*"`：过滤符合表达式的标签。
- `git show <tag>`：展示标签信息和与之对应的提交信息。

### 创建标签

- **轻量标签**：仅仅是对某个提交的引用，不建议使用。
- **附注标签**：是一个完整的对象，可以携带作者、日期、信息等，创建时使用 `-a` 选项。

```bash
# 在当前提交上打标签
git tag -a v1.0.0 -m "发布第一个正式版本"

# 在历史提交上打标签（在最后指定提交的 hash 值）
git tag -a v0.9.0 -m "内测版本" a1b2c3d
```

### 标签共享

标签必须手动推送，`git push` 命令默认不推送标签信息。

```bash
git push origin v1.0.0       # 推送指定的标签到远程
git push origin --tags       # 一次推送全部标签
```

## 11. 分支管理 (git branch)

Git 的分支本质上是一个指向提交对象的可变指针。

### 基础操作

- `git branch`：列出所有本地分支。
- `git branch -v`：查看本地每一个分支的最后一次提交。
- `git branch <branch_name>`：创建新分支（本质是创建一个指向当前分支最新提交对象的指针）。
- `git branch -d <branch_name>`：删除分支。
- `git branch -vv`：查看本地分支跟踪的远程分支。

### 分支切换

- `git checkout <branch_name>`：切换到已经存在的分支。
- `git checkout -b <branch_name>`：创建并切换到新分支。
- `git switch <branch_name>`：新版切换分支命令（推荐）。

### 分支合并

- `git merge <branch_name>`：将其他分支合并到当前分支。
- `git merge --squash <branch_name>`：将其他分支合并到当前分支，但是**只保留一次合并提交**，执行之后需要手动 `commit`。

### 分支冲突

出现冲突的文件片段会用 `<<<<<<<` 和 `>>>>>>>` 包围起来，需要手动解决冲突，保留由 `=======` 分割的两部分中需要的代码。

### 跟踪与拉取远程分支

- `git fetch origin`：同步远程分支信息到本地快照。
- `git switch <branch_name>`：如果本地没有该分支，但远程有同名分支，会自动建立同名本地分支并设置跟踪。
- `git switch -c <local_branch> --track <origin/remote_branch>`：从远程分支引用创建自定义名称的本地分支。
- `git pull`：等价于 `git fetch + git merge`。


⚠️ **注意**：创建跟踪分支时用的都是远程分支的本地快照，所以在创建前得先 `fetch` 才能根据远程最新提交创建。
更常用的是 fetch 之后，直接 `checkout` 或 `switch` 到 `<分支名>`，这个操作会直接创建和远程分支同名的本地分支。


### 推送分支

- `git push <remote> <branch>`：显式指定将本地某个分支推送到某个远程仓库。
- `git push`：推送到当前分支的上游仓库分支，需要建立跟踪关系。
    - 首次推送使用 `git push -u <remote> <branch>`：`-u` 可以建立跟踪关系，以后只需输入 `git push` 即可。

## 12. 变基 (git rebase)
改变（当前分支的）基底
### 基础变基

`git rebase <branch>`：将当前分支上的提交应用到 `<branch>` 的最新提交之后，并将当前分支指针移动到新应用的提交上。
变基相比于合并操作，提交的历史更加简洁，**可以将原本的分叉变为线性**。
```
初始分支状态：
      D---E  (feature 指针在这里)
     /
A---B---C3 (master 指针在这里)

在feature分支上执行 git rebase master 后：
             D'---E' (feature 指针移动到了这里)
            /
A---B---C3 (master)
```

💡 **最佳实践**：一般来说，在向 master 进行 merge 代码之前，应该先在开发分支上执行 `git rebase master`，解决冲突后再发起合并请求（MR/PR）。

- `git rebase --onto <branch1> <branch2> <branch3>`：高级用法。将在 branch3 但是不在 branch2 的提交重用到 branch1 上，然后将 branch3 分支的指针指向重用后的最新提交上。

```
初始分支状态：                    
      D---E  (b3)
     /
A---B---C (b2)
     \
      X (b1)

执行git rebase --onto b1 b2 b3 后：
      D---E (原来的 b3 提交，现在成了孤儿)
     /
A---B---C (b2)
     \
      X---D'---E' (b3 指针移动到了这里！)
```

- `git rebase <basebranch> <topicbranch>`：将 topicbranch 变基到 basebranch 上。


⚠️ **黄金法则**：**不要在已经推送到远程的公共分支上进行变基！** 这会重写提交历史，导致其他协作者产生意外的冲突。


### 变基拉取

`git pull --rebase`：在拉取远程的时候使用 rebase 代替 merge，这样可以避免出现无意义的分叉提交。

- 配置此项可使 `git pull` 默认启用 `--rebase` 参数：
`git config --global pull.rebase true`

### 交互式变基

`git rebase -i HEAD~n`：将当前分支的最近 n 个提交进行整理（如合并、修改信息）。执行后会打开编辑器，按照**上旧下新**顺序列出最近的提交。通过标记每一次请求来决定是保留还是压缩到上一个提交

标记：

- `pick (p)`：标记的基础提交，作为下边的 squash 的基础提交。
- `squash (s)`：标记当前提交为压缩到上边的一个基础提交里，上边必须有一个 pick 作为基础。
- `reword (r)`：保留提交，但修改提交信息。
- `fixup (f)`: 标记当前提交为压缩到上边的一个基础提交里，复用上一个基础提交的提交信息

当前工作区先后有c1 c2 c3 c4 c5 5个提交，想要将5个提交合并为3个，合并前现状如下：
```
e5f6g7h (HEAD -> feature/login) c5: 删除调试用的console.log
d4e5f6g c4: 优化登录按钮样式
c3d4e5f c3: 修复密码校验的拼写错误
b2c3d4e c2: 添加密码校验逻辑
a1b2c3d c1: 实现登录页面骨架
```
执行`git rebase -i HEAD~5`，会打开默认编辑器，按照上旧下新的顺序展示提交
```
pick a1b2c3d c1: 实现登录页面骨架
pick b2c3d4e c2: 添加密码校验逻辑
pick c3d4e5f c3: 修复密码校验的拼写错误
pick d4e5f6g c4: 优化登录按钮样式
pick e5f6g7h c5: 删除调试用的console.log
```
修改标记，把c3合并到c2, c5合并到c4
```
pick a1b2c3d c1: 实现登录页面骨架
pick b2c3d4e c2: 添加密码校验逻辑
s c3d4e5f c3: 修复密码校验的拼写错误
r d4e5f6g c4: 优化登录按钮样式
f e5f6g7h c5: 删除调试用的console.log
```
保存内容退出后会再次弹出两次编辑器编辑c2和c3合并后的提交信息以及c4的reword指令带来的提交信息修改
```
改前：c1 → c2 → c3 → c4 → c5  （5个）

操作：p    p    s     r    f

改后：c1 → c2'(含c3) → c4'(含c5，信息已改)  （3个）
```

## 13. 其他常用神仙命令

| 命令 | 作用说明 |
| ------ |------ |
| `git stash` | 暂存当前工作区修改（用于临时切换分支处理紧急 Bug） |
| `git stash pop` | 恢复最近一次 stash 的修改并删除 stash 记录 |
| `git cherry-pick <commit>` | 将某次特定的提交“复制”并应用到当前分支 |
| `git reflog` | 查看 HEAD 指针移动历史（**找回“丢失”或误 reset 的提交的神器**） |

---
