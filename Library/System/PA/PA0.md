# First Exploration with GNU/Linux
In some Linux distributions, executing the command above may give an error message:

```
-bash: poweroff: command not found
```

This error is due to the property of the `poweroff` command - it is a system administration command. In such Linux distribution, executing this command requires superuser privilege.

Therefore, to shut down the system, you should first switch to the root account:

```
su -
```

Enter the root password you set during the installation. Note that the password is not shown in the terminal to avoid password leaks. If the password is correct, you will see the prompt changes:

```
root@hostname:/home/username#
```

The last character is `#`, instead of `$` before you executing `su -`. `#` is the indicator of root account. Now execute `poweroff` command again, you will find that the command is executed successfully.

# More Exploration
## Learning to use basic tools
[Here](https://linuxconfig.org/gdb-debugging-tutorial-for-beginners) is a small tutorial for GDB

---
# How to Debug Bash Scripts
## How to use Bash xtrace option
```
$ bash -x <scriptname>
```

This tells Bash to show us what each statement looks like after evaluation, just before it is executed. We’ll see an example of this in action shortly, but first let’s contrast `-x` with its opposite `-v`, which shows each line before it is evaluated instead of after. Options can be combined and by using both `-x` and `-v` you can see what statements look like before and after variable substitutions take place.

![[Pasted image 20241101145555.png]]

Notice how using `-x` and `-v` options together allows us to see the original if statement before the `$USER` variable is expanded, thanks to the `-v` option. We also see on the line beginning with a plus sign what the statement looks like again after the substitution, which shows us the actual values compared inside the `if` statement. In more complicated examples this can be quite useful.

## How to use other Bash options
用于调试的 Bash 选项在默认情况下是关闭的，但是一旦通过使用 set 命令打开它们，它们将一直保持开启状态，直到显式关闭为止。如果不确定启用了哪些选项，可以检查 $- 变量以查看所有变量的当前状态。

![[Pasted image 20241101145951.png]]
https://linuxconfig.org/how-to-debug-bash-scripts
### tobe done



---

`git@github.com:` 和 `https://github.com/` 是两种不同的 Git 仓库访问协议，各自有不同的使用场景和优缺点：

### 1. SSH 协议 (`git@github.com:`)
- **格式**：`git@github.com:username/repo.git`
- **认证方式**：使用 SSH 密钥进行身份验证，通常不需要输入用户名和密码。
- **优点**：
  - 更加安全，适合长期使用。
  - 可以免去每次推送或拉取时输入密码的麻烦。
- **缺点**：
  - 需要事先配置 SSH 密钥，并将公钥添加到 GitHub 账户。

### 2. HTTPS 协议 (`https://github.com/`)
- **格式**：`https://github.com/username/repo.git`
- **认证方式**：使用用户名和密码（或令牌）进行身份验证。
- **优点**：
  - 简单易用，尤其是对于新用户或临时访问。
  - 不需要配置 SSH 密钥，直接使用账户凭证即可。
- **缺点**：
  - 每次推送或拉取时，可能需要输入用户名和密码，或者使用 Personal Access Token。
  - 如果不使用 Git Credential Manager 等工具，可能会影响使用体验。

### 总结
- 使用 SSH 协议更适合长期项目和开发工作，而 HTTPS 协议则更方便于临时访问或新手用户。选择哪种方式取决于个人偏好和安全需求。

---

# Getting Source Code for PAs

The workflow above shows how you will use branch in PAs:

- before starting a new PA, new a branch `pa?` and check out to it
- coding in the branch `pa?` (this will introduce lot of modifications)
- after finish the PA, merge the branch `pa?` into `master`, and check out back to `master`


---
# git 快速入门
![[Pasted image 20241102102529.png]]

在 WSL（Windows Subsystem for Linux）上创建一个新用户，可以按照以下步骤进行：

### 1. 打开 WSL 终端

首先，打开你安装的 WSL 终端（例如 Ubuntu、Debian 等）。

### 2. 更新软件包列表（可选）

在创建用户之前，最好更新软件包列表：

```bash
sudo apt update
```

### 3. 创建新用户

使用 `adduser` 命令来创建新用户。假设你要创建一个名为 `newuser` 的用户，运行以下命令：

```bash
sudo adduser newuser
```

系统会提示你输入新用户的密码和其他信息（如全名、房间号等）。根据提示输入相应信息。

### 4. 添加用户到 sudo 组（可选）

如果你希望新用户具有超级用户权限，可以将其添加到 `sudo` 组：

```bash
sudo usermod -aG sudo newuser
```

### 5. 切换到新用户

创建完成后，你可以使用以下命令切换到新用户：

```bash
su - newuser
```

### 6. 验证新用户

切换用户后，你可以使用以下命令验证新用户的创建：

```bash
whoami
```

这将显示当前用户的用户名。

