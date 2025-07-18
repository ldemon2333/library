## Git Cherry-pick 教学

`git cherry-pick` 是 Git 中一个非常实用的命令，它允许你**选择性地将一个或多个提交（commit）从一个分支“复制”到另一个分支**。不同于 `git merge` 或 `git rebase` 它们通常处理一系列提交，`cherry-pick` 让你能精确地挑选并应用特定的提交。

想象一下这个场景：你在一个特性分支上工作，但不小心提交了一个 Bug 修复，这个修复其实应该在主分支上立即发布。或者，你需要将某个 Bug 修复从一个旧的发布分支带到当前的开发分支。这时，`cherry-pick` 就派上用场了。

---

### `git cherry-pick` 的工作原理

当你在一个分支上执行 `git cherry-pick <commit_hash>` 时，Git 会做以下几件事：

1. **找到指定提交的更改：** Git 会找出 `<commit_hash>` 所引入的所有修改（文件增删、代码改动等）。
    
2. **在当前分支上重新应用：** Git 会尝试将这些修改应用到你当前所在的分支上。
    
3. **创建一个新提交：** 如果应用成功，Git 会在当前分支上创建一个**新的提交**，这个新提交包含了被 cherry-pick 的提交的所有更改。这个新提交会有不同的哈希值，因为它的父提交和提交时间可能与原始提交不同。
    

**示例：**

假设你有以下提交历史：

```
A -- B -- C (main)
     \
      D -- E -- F (feature)
```

现在你想把 `feature` 分支上的提交 `E`（假设它是一个重要的 Bug 修复）应用到 `main` 分支上。

1. 首先，**切换到目标分支** (`main`)：
    
    Bash
    
    ```
    git checkout main
    ```
    
2. 然后，**cherry-pick 提交 `E`**：
    
    Bash
    
    ```
    git cherry-pick E_hash_value
    ```
    

执行后，你的历史会变成这样：

```
A -- B -- C -- E' (main)
     \
      D -- E -- F (feature)
```

注意，`E'` 是一个全新的提交，它包含了 `E` 中的所有更改，但现在它位于 `main` 分支上。

---

### `git cherry-pick` 的基本用法

Bash

```
git cherry-pick <commit_hash>
```

- `<commit_hash>` 是你想要复制的**目标提交的完整哈希值或其前几位**。
    

**例子：**

Bash

```
# 假设你在 main 分支上
git checkout main

# 从 feature 分支上挑选一个 Bug 修复提交 (哈希值为 a1b2c3d4)
git cherry-pick a1b2c3d4
```

---

### 常用选项

#### 1. Cherry-pick 多个提交

你可以一次性 cherry-pick 多个提交。Git 会按照你指定的顺序依次应用它们。

Bash

```
git cherry-pick <commit_hash1> <commit_hash2> <commit_hash3>
```

#### 2. Cherry-pick 一个范围的提交

你可以指定一个范围来 cherry-pick 多个连续的提交。这类似于 `revert` 命令的范围用法。

Bash

```
git cherry-pick <起始提交哈希值>..<结束提交哈希值>
```

**注意：** 这里的范围是“不包含起始提交，包含结束提交”。也就是说，它会 cherry-pick `起始提交哈希值` **之后**的所有提交，直到 `结束提交哈希值` 为止（包括 `结束提交哈希值`）。

例如，如果你想 cherry-pick 从 `commit-X` **之后**到 `commit-Y` **（包括 `commit-Y`）**的所有提交：

Bash

```
git cherry-pick X_hash..Y_hash
```

如果你想包含 `commit-X` 本身，可以使用 `^` 符号：

Bash

```
git cherry-pick X_hash^..Y_hash
```

#### 3. Cherry-pick 不立即提交 (`-n` 或 `--no-commit`)

如果你想将 cherry-pick 的更改应用到工作区和暂存区，但不立即创建一个新的提交，可以使用 `-n` 选项。这允许你在提交之前对更改进行额外的修改。

Bash

```
git cherry-pick -n <commit_hash>
```

应用后，你可以进行额外的修改，然后 `git add` 和 `git commit`。

#### 4. 编辑提交信息 (`-e` 或 `--edit`)

在 cherry-pick 成功后，Git 会弹出一个编辑器让你修改新创建提交的提交信息。如果你想强制弹出编辑器，即使没有冲突，也可以使用 `-e`。

Bash

```
git cherry-pick -e <commit_hash>
```

---

### 冲突解决

当 cherry-pick 的提交与目标分支的当前状态发生冲突时，Git 会暂停操作，并提示你解决冲突。解决冲突的步骤与 `merge` 或 `rebase` 类似：

1. **解决冲突：** 打开冲突文件，手动编辑，删除冲突标记（`<<<<<<<`, `=======`, `>>>>>>>`），并保留你想要的代码。
    
2. **暂存解决后的文件：** `git add <冲突文件>`。
    
3. **继续 cherry-pick：** `git cherry-pick --continue`。
    
4. **取消 cherry-pick：** 如果你决定放弃当前的 cherry-pick 操作：`git cherry-pick --abort`。
    

---

### 何时使用 `git cherry-pick`

- **将 Bug 修复从一个分支带到另一个分支：** 比如，你在 `develop` 分支上发现并修复了一个 Bug，但这个 Bug 也存在于已发布的 `main` 分支上。你可以 cherry-pick 这个修复提交到 `main` 分支。
    
- **选择性地应用特性：** 你在一个大的特性分支上工作，但其中包含了几个独立的小功能。你可能只想将其中一个功能（由一个或几个提交组成）提前发布。
    
- **整理提交历史：** 在某些复杂的工作流中，你可能需要将某个旧的、重要的提交重新应用到新的开发线上。
    

---

### `git cherry-pick` 的注意事项

- **重复的提交：** `cherry-pick` 会在目标分支上创建一个**新的提交**，即使这个提交的内容和原始提交完全相同。这意味着你的代码库中可能存在相同内容但不同哈希值的提交。
    
- **不改变原始历史：** `cherry-pick` 不会修改原始提交或其所在分支的历史。
    
- **潜在的冲突：** 由于 `cherry-pick` 是将一个提交独立地应用，如果该提交所依赖的上下文（即其父提交的内容）在目标分支上发生了变化，就很容易引发冲突。
    
- **谨慎使用：** 尽管 `cherry-pick` 很方便，但过度使用可能会导致历史变得混乱，难以追踪。通常，对于整个特性的集成，`merge` 或 `rebase` 是更好的选择。只有当你需要精确地挑选少数提交时，才考虑使用 `cherry-pick`。
    

---

希望这个详细的 `git cherry-pick` 教学能帮助你更好地理解和使用这个强大的 Git 命令！你还有其他想了解的 Git 功能吗？