## Git Rebase 详细教学

`git rebase` 是 Git 中一个强大的命令，它能让你**重写提交历史**。与 `git merge` 不同，`rebase` 不会创建新的合并提交，而是通过将一系列提交“移动”到新的基点上，来形成一个更线性、更“干净”的提交历史。

理解 `rebase` 可能会有点复杂，但一旦掌握，它能让你的项目历史看起来更加整洁和专业。

---

### `git rebase` 的基本原理

想象你有以下提交历史：

```
A -- B -- C (main)
     \
      D -- E -- F (feature)
```

你想将 `feature` 分支上的更改合并到 `main` 分支上。

**如果使用 `git merge feature`：**

```
A -- B -- C -- G (main, merge commit)
     \        /
      D -- E -- F (feature)
```

`git merge` 会创建一个新的合并提交 `G`，记录了 `feature` 分支的合并点。这保留了完整的历史，包括分支的创建和合并。

**如果使用 `git rebase main`（在 `feature` 分支上）：**

1. Git 会找到 `feature` 分支和 `main` 分支的共同祖先（在这个例子中是 `B`）。
    
2. 它会“剪切”掉 `feature` 分支上从共同祖先之后的所有提交（D, E, F）。
    
3. 它会将这些剪切下来的提交，**依次**重新应用到 `main` 分支的最新提交 `C` 的**上方**。
    

结果会是这样：

```
A -- B -- C -- D' -- E' -- F' (feature)
           ^
           |
          (main)
```

注意，`D'`, `E'`, `F'` 是原始提交 `D`, `E`, `F` 的**新版本**。它们的提交哈希值会改变，因为它们的父提交发生了变化。现在，`feature` 分支看起来就像是从 `main` 的最新提交 `C` 上直接创建的。

---

### `git rebase` 的优点和缺点

#### 优点：

- **更清晰的提交历史：** 提交历史变成了一条直线，没有了合并提交，更容易阅读和理解。
    
- **消除不必要的合并提交：** 尤其是在个人开发分支上，可以避免创建大量的“噪音”合并提交。
    
- **准备推送：** 在将本地更改推送到远程之前，`rebase` 可以确保你的本地分支基于远程的最新版本，从而减少推送时的冲突。
    

#### 缺点：

- **改变提交哈希：** `rebase` 会重写提交历史，这意味着被 rebase 的提交的哈希值会改变。
    
- **不适用于已共享的提交：** **这是最重要的一点！** 永远不要对已经推送到共享远程仓库的提交进行 `rebase`。如果其他人已经基于你 rebase 之前的提交进行了工作，那么你的 `rebase` 会导致他们的历史与你的不匹配，从而引发严重的冲突和混乱。这被称为“黄金法则”：**不要 rebase 已经发布（pushed）的提交。**
    
- **更复杂的冲突解决：** 如果在 `rebase` 过程中遇到冲突，你需要逐个解决每个冲突的提交，这可能比 `merge` 的一次性冲突解决更复杂。
    

---

### `git rebase` 的常用场景

#### 1. 合并你的本地更改到远程最新版本

这是最常见的 `rebase` 用途。当你在自己的特性分支上工作时，其他人可能已经向 `main` 分支提交了新的内容。为了让你的特性分支跟上 `main` 的最新进展，你可以在你的特性分支上执行 `rebase`：


```
# 1. 切换到你的特性分支
git checkout my-feature-branch

# 2. 确保你的本地 main 分支是最新的（可选，但推荐）
git checkout main
git pull origin main
git checkout my-feature-branch

# 3. 将你的特性分支 rebase 到 main 分支上
git rebase main
```

这会将你的 `my-feature-branch` 上的提交“移动”到 `main` 分支的最新提交之后。

#### 2. 交互式 rebase (`git rebase -i`)

交互式 `rebase` 是 `rebase` 的一个非常强大的功能，它允许你在 `rebase` 过程中对提交进行更多的操作，例如：

- **squash (压缩)：** 将多个提交合并成一个。
    
- **fixup (合并但不保留提交信息)：** 与 squash 类似，但会丢弃被合并提交的提交信息。
    
- **reword (重写提交信息)：** 修改提交信息。
    
- **edit (编辑提交)：** 在 rebase 过程中停止，允许你修改该提交。
    
- **drop (删除)：** 完全删除某个提交。
    
- **reorder (重新排序)：** 改变提交的顺序。
    

**用法：**

Bash

```
git rebase -i <commit_hash>
```

`<commit_hash>` 是你想要重写历史的**基点**提交。Git 会列出从这个提交之后的所有提交，并弹出一个文本编辑器，其中包含类似以下内容：

```
pick f7f3f6d Commit A
pick 3fa4a9b Commit B
pick 8c2d1e0 Commit C

# Rebase 2d2c803..8c2d1e0 onto 2d2c803 (3 commands)
#
# Commands:
# p, pick <commit> = use commit
# r, reword <commit> = use commit, but edit the commit message
# e, edit <commit> = use commit, but stop for amending
# s, squash <commit> = use commit, but meld into previous commit
# f, fixup <commit> = like "squash", but discard this commit's log message
# x, exec <command> = run command (the rest of the line) for each commit
# b, break = stop here (continue rebase later with 'git rebase --continue')
# d, drop <commit> = remove commit
# l, label <label> = label current HEAD with a name
# t, reset <label> = reset HEAD to a label
# m, merge [-C <commit> | -c <commit>] <label> [# <oneline>]
# .       create a merge commit
#
# These lines can be re-ordered; they are executed from top to bottom.
#
# If you remove a line here THAT COMMIT WILL BE LOST.
#
# However, if you remove everything, the rebase will be aborted.
#
```

你可以修改 `pick` 为其他命令，并重新排列提交的顺序。保存并关闭文件后，Git 将按照你的指示执行 `rebase`。

**交互式 rebase 的常见场景：**

- 在推送到远程之前，清理本地提交历史，将小的、不重要的提交压缩成一个有意义的提交。
    
- 修正之前提交中的错误（例如，修改提交信息或添加遗漏的文件）。
    

---

### `git rebase` 冲突解决

当 `rebase` 遇到冲突时，它会暂停执行，并在命令行中提示你。冲突解决的过程与 `merge` 冲突类似：

1. **解决冲突：** 打开冲突文件，手动编辑，删除冲突标记（`<<<<<<<`, `=======`, `>>>>>>>`），并保留你想要的代码。
    
2. **暂存解决后的文件：** `git add <冲突文件>`。
    
3. **继续 `rebase`：** `git rebase --continue`。
    
4. **取消 `rebase`：** 如果你决定放弃当前的 `rebase` 操作，可以使用 `git rebase --abort`。
    

**注意：** 如果你使用了 `squash` 或 `fixup` 命令，Git 可能会在完成 `rebase` 后提示你编辑新的合并提交信息。

---

### `git rebase` 与 `git pull --rebase`

git pull 默认是 git fetch 和 git merge 的组合。

git pull --rebase 则是 git fetch 和 git rebase 的组合。

如果你经常在本地分支上工作，并且希望在拉取远程更新时保持线性的提交历史，你可以将 `pull.rebase` 配置为 `true`：

Bash

```
git config --global pull.rebase true
```

这样，每次你执行 `git pull` 时，Git 都会自动尝试进行 `rebase` 而不是 `merge`。

---

### 总结与最佳实践

- **理解 `rebase` 会重写历史：** 这是最核心的概念。
    
- **不要 rebase 已发布的提交：** 这是**黄金法则**。只有当你确定这些提交只存在于你本地，并且没有其他人基于它们进行工作时，才可以使用 `rebase`。
    
- **经常 rebase：** 如果你在一个长期运行的特性分支上工作，定期将其 rebase 到 `main` 分支的最新状态，可以帮助你及早发现并解决冲突，而不是等到最后才处理一个巨大的冲突。
    
- **谨慎使用交互式 rebase：** 交互式 rebase 非常强大，但也很容易出错。在不确定时，先备份你的分支，或者在一个测试仓库中练习。
    
- **合并 vs. rebase：**
    
    - **选择 `merge`：** 当你需要保留完整的历史记录，包括分支的创建和合并点，或者当你在多人协作的项目中合并共享分支时。
        
    - **选择 `rebase`：** 当你希望拥有一个干净、线性的提交历史，尤其是在你将本地更改推送到远程之前，或者在清理个人开发分支时。
        

`git rebase` 是一个进阶的 Git 命令，熟练掌握它能显著提升你的 Git 使用效率和代码库的整洁度。但请务必记住它的局限性，特别是关于已发布提交的“黄金法则”。