[命令行的艺术](https://github.com/jlevy/the-art-of-command-line/blob/master/README-zh.md)[[命令行的艺术]]
# Topic 1: The Shell

## What is the shell?

A shell is a command-line interface that allows users to interact with the operating system. It provides a way to execute commands, run scripts, and manage system resources. Shells can be text-based (like Bash or Zsh) or graphical. They serve as an intermediary between the user and the operating system, translating user commands into actions performed by the system.

## Using the shell

When you launch your terminal, you will see a *prompt* that often looks a little like this:
```shell
missing:~$ 
```
This is the main textual interface to the shell. It tells you that you are on the machine `missing` and that your “current working directory”, or where you currently are, is `~` (short for “home”). The `$` tells you that you are not the root user. At this prompt you can type a *command*, which will then be interpreted by the shell. The most basic command is to execute a program:

```sh
missing:~$ date
Fri 10 Jan 2020 11:49:31 AM EST
missing:~$ 
```

Here, we executed the `date` program, which (perhaps unsurprisingly) prints the current date and time. 

```sh
missing:~$ echo hello
hello
```

In this case, we told the shell to execute the program `echo` with the argument `hello`. The `echo` program simply prints out its arguments. The shell parses the command by splitting it by whitespace, and then runs the program indicated by the first word, supplying each subsequent word as an argument that the program can access. If you want to provide an argument that contains spaces or other special characters (e.g., a directory named “My Photos”), you can either quote the argument with `'` or `"` (`"My Photos"`), or escape just the relevant characters with `\` (`My\ Photos`).

But how does the shell know how to find the `date` or `echo` programs? Well, the shell is a programming environment, just like Python or Ruby, and so it has variables, conditionals, loops, and functions. When you run commands in your shell, you are really writing a small bit of code that your shell interprets. If the shell is asked to execute a command that doesn’t match one of its programming keywords, it consults an *environment variable* called `$PATH` that lists which directories the shell should search for programs when it is given a command:

```sh
missing:~$ echo $PATH
/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
missing:~$ which echo
/bin/echo
missing:~$ /bin/echo $PATH
/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
```

When we run the `echo` command, the shell sees that it should execute the program `echo`, and then searches through the `:`-separated list of directories in `$PATH` for a file by that name. When it finds it, it runs it (assuming the file is *executable*). We can find out which file is executed for a given program name using the `which` program. We can also bypass `$PATH` entirely by giving the *path* to the file we want to execute.

## Navigating in the shell

A path on the shell is a delimited list of directories; separated by `/` on Linux and macOS and `\` on Windows. On Linux and macOS, the path `/` is the “root” of the file system, under which all directories and files lie, whereas on Windows there is one root for each disk partition (e.g., `C:\`). A path that starts with `/` is called an *absolute* path. Any other path is a *relative* path. Relative paths are relative to the current working directory, which we can see with the `pwd` command and change with the `cd` command. In a path, `.` refers to the current directory, and `..` to its parent directory:

```sh
missing:~$ pwd
/home/missing
missing:~$ cd /home
missing:/home$ pwd
/home
missing:/home$ cd ..
missing:/$ pwd
/
missing:/$ cd ./home
missing:/home$ pwd
/home
missing:/home$ cd missing
missing:~$ pwd
/home/missing
missing:~$ ../../bin/echo hello
hello
```

Notice that our shell prompt kept us informed about what our current working directory was.

In general, when we run a program, it will operate in the current directory unless we tell it otherwise. For example, it will usually search for files there, and create new files there if it needs to.

For example, `ls --help` tells us:

```sh
  -l                         use a long listing format
missing:~$ ls -l /home
drwxr-xr-x 1 missing  users  4096 Jun 15  2019 missing
```

This gives us a bunch more information about each file or directory present. First, the `d` at the beginning of the line tells us that `missing` is a directory. Then follow three groups of three characters (`rwx`). These indicate what permissions the owner of the file (`missing`), the owning group (`users`), and everyone else respectively have on the relevant item. A `-` indicates that the given principal does not have the given permission. Above, only the owner is allowed to modify (`w`) the `missing` directory (i.e., add/remove files in it). To enter a directory, a user must have “search” (represented by “execute”: `x`) permissions on that directory (and its parents). To list its contents, a user must have read (`r`) permissions on that directory. For files, the permissions are as you would expect. Notice that nearly all the files in `/bin` have the `x` permission set for the last group, “everyone else”, so that anyone can execute those programs.

If you ever want *more* information about a program’s arguments, inputs, outputs, or how it works in general, give the `man` program a try. It takes as an argument the name of a program, and shows you its *manual page*. Press `q` to exit.

```sh
missing:~$ man ls
```

`chmod` 是一个 Unix 和类 Unix 操作系统中的命令，用于更改文件或目录的权限。权限控制在 Unix 系统中非常重要，它可以确保文件和目录的安全性和正确的访问权限。以下是 `chmod` 命令的详细用法及示例。

### 文件权限的基本概念

Unix 文件系统的权限分为三类：
1. **用户 (User)**：文件的所有者。
2. **组 (Group)**：文件所属的用户组。
3. **其他 (Others)**：除了用户和组之外的所有其他用户。

每个类别都有三种权限：
1. **读 (Read, r)**：允许读取文件内容。
2. **写 (Write, w)**：允许修改文件内容。
3. **执行 (Execute, x)**：允许执行文件（如果文件是可执行程序或脚本）。

这些权限用三个三位的八进制数字来表示，每个数字代表一个类别的权限。例如，`rwxr-xr--` 表示：
- 用户：读、写、执行 (7 = rwx)
- 组：读、执行 (5 = r-x)
- 其他：读 (4 = r--)

### 使用 `chmod` 命令

#### 符号模式

使用符号模式设置权限：

- `u` 表示用户 (user)
- `g` 表示组 (group)
- `o` 表示其他 (others)
- `a` 表示所有 (all)
- `+` 表示添加权限
- `-` 表示移除权限
- `=` 表示设置权限

##### 示例：
```sh
chmod u+x file.txt       # 给用户添加执行权限
chmod g-w file.txt       # 移除组的写权限
chmod o=r file.txt       # 设置其他用户只读权限
chmod a+rw file.txt      # 给所有用户添加读写权限
```

#### 八进制模式

使用八进制数设置权限：

- `4` 表示读 (r)
- `2` 表示写 (w)
- `1` 表示执行 (x)

这些数值可以相加来组合权限。例如：
- `7` 表示读、写、执行 (4 + 2 + 1 = rwx)
- `6` 表示读、写 (4 + 2 = rw-)
- `5` 表示读、执行 (4 + 1 = r-x)
- `4` 表示只读 (4 = r--)

##### 示例：
```sh
chmod 755 file.txt       # 用户: rwx, 组: r-x, 其他: r-x
chmod 644 file.txt       # 用户: rw-, 组: r--, 其他: r--
chmod 600 file.txt       # 用户: rw-, 组: ---, 其他: ---
chmod 777 file.txt       # 所有用户都有读、写、执行权限
```

### 递归更改目录权限

使用 `-R` 选项可以递归地更改目录及其子目录和文件的权限：
```sh
chmod -R 755 directory   # 递归设置目录及其内容的权限
```

### 示例

#### 将文件设置为只读
```sh
chmod 444 file.txt       # 所有用户只读
chmod a-w file.txt       # 移除所有用户的写权限
```

#### 给脚本添加执行权限
```sh
chmod +x script.sh       # 添加执行权限
chmod 755 script.sh      # 设置用户: rwx, 组: r-x, 其他: r-x
```

#### 设置目录权限
```sh
chmod 755 mydir          # 设置用户: rwx, 组: r-x, 其他: r-x
chmod -R 755 mydir       # 递归设置目录及其内容的权限
```

## Connecting programs

In the shell, programs have two primary “streams” associated with them: their input stream and their output stream. When the program tries to read input, it reads from the input stream, and when it prints something, it prints to its output stream. Normally, a program’s input and output are both your terminal. That is, your keyboard as input and your screen as output. However, we can also rewire those streams!

The simplest form of redirection is `< file` and `> file`. These let you rewire the input and output streams of a program to a file respectively:

```sh
missing:~$ echo hello > hello.txt
missing:~$ cat hello.txt
hello
missing:~$ cat < hello.txt
hello
missing:~$ cat < hello.txt > hello2.txt
missing:~$ cat hello2.txt
hello
```

Demonstrated in the example above, `cat` is a program that con`cat`enates files. When given file names as arguments, it prints the contents of each of the files in sequence to its output stream. But when `cat` is not given any arguments, it prints contents from its input stream to its output stream (like in the third example above).

You can also use `>>` to append to a file. Where this kind of input/output redirection really shines is in the use of *pipes*. The `|` operator lets you “chain” programs such that the output of one is the input of another:

```sh
missing:~$ ls -l / | tail -n1
drwxr-xr-x 1 root  root  4096 Jun 20  2019 var
missing:~$ curl --head --silent google.com | grep --ignore-case content-length | cut --delimiter=' ' -f2
219
```

1. **First Command: `ls -l / | tail -n1`**

   ```sh
   missing:~$ ls -l / | tail -n1
   drwxr-xr-x 1 root  root  4096 Jun 20  2019 var
   ```

   - `ls -l /`: Lists the contents of the root directory (`/`) in long format.
   - `| tail -n1`: Pipes the output of `ls -l /` to `tail`, which shows only the last line of the output.

   Explanation of the output:
   - `drwxr-xr-x 1 root  root  4096 Jun 20  2019 var`: This line describes the directory `/var`.
     - `drwxr-xr-x`: Permissions (`d` for directory, `rwx` for owner, `r-x` for group, `r-x` for others).
     - `1`: Number of hard links.
     - `root root`: Owner and group of the directory.
     - `4096`: Size of the directory in bytes.
     - `Jun 20  2019`: Last modification date.
     - `var`: Name of the directory.

2. **Second Command: `curl --head --silent google.com | grep --ignore-case content-length | cut --delimiter=' ' -f2`**

   ```sh
   missing:~$ curl --head --silent google.com | grep --ignore-case content-length | cut --delimiter=' ' -f2
   219
   ```

   - `curl --head --silent google.com`: Sends an HTTP HEAD request to `google.com` silently (without progress meter or error messages), and retrieves only the header information.
   - `grep --ignore-case content-length`: Searches for the line containing `content-length` in the header, ignoring case sensitivity.
   - `cut --delimiter=' ' -f2`: Splits the line by spaces (`-d' '`) and extracts the second field (`-f2`), which is the value of `Content-Length`.

   Explanation of the output:
   - `219`: This is the value of the `Content-Length` header from the HTTP response of `google.com`. It indicates the size of the response body in bytes.

### Summary

- The first command lists the contents of the root directory (`/`) and shows details about the `/var` directory.
- The second command retrieves the `Content-Length` header from an HTTP HEAD request to `google.com`, which indicates the size of the response body (219 bytes in this case).

## A versatile and powerful tool

On most Unix-like systems, one user is special: the “root” user. You may have seen it in the file listings above. The root user is above (almost) all access restrictions, and can create, read, update, and delete any file in the system. You will not usually log into your system as the root user though, since it’s too easy to accidentally break something. Instead, you will be using the `sudo` command. As its name implies, it lets you “do” something “as su” (short for “super user”, or “root”). When you get permission denied errors, it is usually because you need to do something as root. Though make sure you first double-check that you really wanted to do it that way!

One thing you need to be root in order to do is writing to the `sysfs` file system mounted under `/sys`. `sysfs` exposes a number of kernel parameters as files, so that you can easily reconfigure the kernel on the fly without specialized tools. **Note that sysfs does not exist on Windows or macOS.**

For example, the brightness of your laptop’s screen is exposed through a file called `brightness` under

```sh
/sys/class/backlight
```

By writing a value into that file, we can change the screen brightness. Your first instinct might be to do something like:

```sh
$ sudo find -L /sys/class/backlight -maxdepth 2 -name '*brightness*'
/sys/class/backlight/thinkpad_screen/brightness
$ cd /sys/class/backlight/thinkpad_screen
$ sudo echo 3 > brightness
An error occurred while redirecting file 'brightness'
open: Permission denied
```

This error may come as a surprise. After all, we ran the command with `sudo`! This is an important thing to know about the shell. Operations like `|`, `>`, and `<` are done *by the shell*, not by the individual program. `echo` and friends do not “know” about `|`. They just read from their input and write to their output, whatever it may be. In the case above, the *shell* (which is authenticated just as your user) tries to open the brightness file for writing, before setting that as `sudo echo`’s output, but is prevented from doing so since the shell does not run as root. Using this knowledge, we can work around this:

```shell
$ echo 3 | sudo tee brightness
```

Since the `tee` program is the one to open the `/sys` file for writing, and *it* is running as `root`, the permissions all work out. You can control all sorts of fun and useful things through `/sys`, such as the state of various system LEDs (your path might be different):

```sh
$ echo 1 | sudo tee /sys/class/leds/input6::scrolllock/brightness
```

Let's delve deeper into why the command `sudo echo 3 > brightness` failed and how to correctly handle such situations in Unix-like systems.

### Understanding the Issue

The command you attempted was:

```sh
sudo echo 3 > brightness
```

However, this didn't work as expected due to the way Unix shells handle redirections (`>`). Here’s what happened step-by-step:

1. **Shell Redirection**: The `>` symbol in Unix shell commands is used for output redirection. In this case, `sudo echo 3 > brightness` instructs the shell to redirect the output of `echo 3` to the file named `brightness`.

2. **Shell Execution Order**: Before executing the command, the shell processes the redirection (`>`). This means the shell, running with your user's permissions (even though `sudo` is used), tries to open and write to the `brightness` file in `/sys/class/backlight/thinkpad_screen`.

3. **Permission Denied**: The error `Permission denied` occurs because the shell, which is authenticated as your user, does not have permission to write to the `brightness` file. Even though `sudo` elevates the privileges for the `echo` command itself, it does not affect the redirection operation, which is handled by the shell.

### Correct Approach: Using `tee` with `sudo`

To work around this issue and correctly write to the `brightness` file with elevated privileges, you can use `tee`, which can be run with `sudo` to handle the file writing part:

```sh
echo 3 | sudo tee /sys/class/backlight/thinkpad_screen/brightness
```

#### Explanation:

- `echo 3`: Outputs `3` to standard output.
- `|`: Pipes the output of `echo 3` to the next command.
- `sudo tee /sys/class/backlight/thinkpad_screen/brightness`: `tee` reads from standard input and writes to standard output and the specified file (`brightness` in this case). The `sudo` command ensures that `tee` runs with superuser privileges, allowing it to write to `/sys/class/backlight/thinkpad_screen/brightness`.

### Why `tee` Works:

- **Shell Independence**: Unlike redirection (`>`), which is handled by the shell, `tee` is a separate command that runs with elevated privileges (`sudo`). It avoids the permission issue because `sudo` is applied to the entire command (`tee` and the file operation), not just the `echo` command.

### Summary:

The `Permission denied` error occurred because the shell attempted to perform the file redirection (`>`) operation with your user's permissions, even though `sudo` was used for the `echo` command. Using `sudo tee` is a reliable workaround in Unix-like systems when you need to write to files that require superuser privileges. It ensures that both the command and the file operation are executed with elevated permissions, thereby avoiding permission issues like the one encountered with `sudo echo >`.

## Exercises

The provided script consists of two lines:

```sh
#!/bin/sh
curl --head --silent https://missing.csail.mit.edu
```

### Explanation of Each Line

1. **Shebang Line (`#!/bin/sh`)**:
   - `#!/bin/sh` is called a "shebang" and is used to specify the interpreter that should be used to execute the script.
   - In this case, `/bin/sh` is specified, which typically points to a POSIX-compliant shell like `dash` on many Unix-like systems.

2. **Curl Command**:
   - `curl --head --silent https://missing.csail.mit.edu` is a command that uses `curl`, a tool to transfer data from or to a server.

### Detailed Breakdown of the `curl` Command:

- **`curl`**: This is the command-line tool used for transferring data with URLs.
  
- **`--head`**: This option tells `curl` to fetch the headers only, without the body of the response. This is useful when you just want to see the metadata of a resource, such as HTTP status code, content type, and other headers, without downloading the entire content.

- **`--silent`**: This option makes `curl` operate in silent mode. It suppresses the progress meter and error messages. This is useful when you want the output to be clean and free of extra information.

- **`https://missing.csail.mit.edu`**: This is the URL from which `curl` will fetch the headers. The URL points to a specific webpage or resource.

### What the Script Does

When you run this script, it performs the following actions:

1. **Interpreter Selection**:
   - The shebang (`#!/bin/sh`) ensures that the script is executed using the `/bin/sh` shell.

2. **Fetch Headers**:
   - The `curl` command sends an HTTP HEAD request to the URL `https://missing.csail.mit.edu`.
   
   - It retrieves and prints the HTTP headers returned by the server for this URL.
   
   - It operates silently, so you won't see the progress meter or error messages.  
   
     

The `grep` command is a tool used to search for specific patterns within text files or input streams. Here are the basics:

### Basic Usage

```sh
grep [options] pattern [file...]
```

### Common Options

- **`-i`**: Ignore case (case-insensitive search).
- **`-v`**: Invert match (show lines that do not match the pattern).
- **`-r`**: Recursively search directories.
- **`-l`**: Show filenames only (list files with matches).
- **`-c`**: Count matching lines.
- **`-n`**: Show line numbers with matches.
- **`--color`**: Highlight matches.

### Examples

1. **Simple Search**:
   ```sh
   grep "pattern" file.txt
   ```

2. **Case-Insensitive Search**:
   ```sh
   grep -i "pattern" file.txt
   ```

3. **Recursive Search**:
   ```sh
   grep -r "pattern" /path/to/dir
   ```

4. **Count Matches**:
   ```sh
   grep -c "pattern" file.txt
   ```

5. **Show Line Numbers**:
   ```sh
   grep -n "pattern" file.txt
   ```

`grep` is useful for finding specific text within files and can handle simple strings or complex patterns using regular expressions.

# Shell Tools and Scripting

# Shell Scripting

To assign variables in bash, use the syntax `foo=bar` and access the value of the variable with `$foo`. Note that `foo = bar` will not work since it is interpreted as calling the `foo` program with arguments `=` and `bar`. In general, in shell scripts the space character will perform argument splitting. This behavior can be confusing to use at first, so always check for that.

Strings in bash can be defined with `'` and `"` delimiters, but they are not equivalent. Strings delimited with `'` are literal strings and will not substitute variable values whereas `"` delimited strings will.

```sh
foo=bar
echo "$foo"
# prints bar
echo '$foo'
# prints $foo
```

Here is an example of a function that creates a directory and `cd`s into it.

```sh
mcd () {
    mkdir -p "$1"
    cd "$1"
}
```

Here `$1` is the first argument to the script/function. Unlike other scripting languages, bash uses a variety of special variables to refer to arguments, error codes, and other relevant variables. Below is a list of some of them. A more comprehensive list can be found [here](https://tldp.org/LDP/abs/html/special-chars.html).

- `$0` - Name of the script
- `$1` to `$9` - Arguments to the script. `$1` is the first argument and so on.
- `$@` - All the arguments
- `$#` - Number of arguments
- `$?` - Return code of the previous command
- `$$` - Process identification number (PID) for the current script
- `!!` - Entire last command, including arguments. A common pattern is to execute a command only for it to fail due to missing permissions; you can quickly re-execute the command with sudo by doing `sudo !!`
- `$_` - Last argument from the last command. If you are in an interactive shell, you can also quickly get this value by typing `Esc` followed by `.` or `Alt+.`

Commands will often return output using `STDOUT`, errors through `STDERR`, and a Return Code to report errors in a more script-friendly manner. The return code or exit status is the way scripts/commands have to communicate how execution went. A value of 0 usually means everything went OK; anything different from 0 means an error occurred.

Exit codes can be used to conditionally execute commands using `&&` (and operator) and `||` (or operator), both of which are [short-circuiting](https://en.wikipedia.org/wiki/Short-circuit_evaluation) operators. Commands can also be separated within the same line using a semicolon `;`. The `true` program will always have a 0 return code and the `false` command will always have a 1 return code. Let’s see some examples

```sh
false || echo "Oops, fail"
# Oops, fail

true || echo "Will not be printed"
#

true && echo "Things went well"
# Things went well

false && echo "Will not be printed"
#

true ; echo "This will always run"
# This will always run

false ; echo "This will always run"
# This will always run
```

Another common pattern is wanting to get the output of a command as a variable. This can be done with *command substitution*. Whenever you place `$( CMD )` it will execute `CMD`, get the output of the command and substitute it in place. For example, if you do `for file in $(ls)`, the shell will first call `ls` and then iterate over those values. A lesser known similar feature is *process substitution*, `<( CMD )` will execute `CMD` and place the output in a temporary file and substitute the `<()` with that file’s name. This is useful when commands expect values to be passed by file instead of by STDIN. For example, `diff <(ls foo) <(ls bar)` will show differences between files in dirs `foo` and `bar`.

下面是关于命令替换（*command substitution*）和进程替换（*process substitution*）的详细说明和示例：

### 命令替换（Command Substitution）

命令替换用于执行命令并将其输出作为变量或参数。在 Bash 中，命令替换可以使用 `$()` 语法来实现。

#### 示例

```sh
# 获取当前目录的文件列表并存储在变量中
files=$(ls)
echo "当前目录的文件：$files"

# 使用命令替换在for循环中迭代文件
for file in $(ls); do
    echo "文件：$file"
done
```

在上面的示例中，`$(ls)` 会执行 `ls` 命令，并将输出替换到相应的位置。

### 进程替换（Process Substitution）

进程替换用于将命令的输出放置在一个临时文件中，并将该文件的名称替换到相应的位置。进程替换使用 `<( CMD )` 语法。

#### 示例

假设有两个目录 `foo` 和 `bar`，我们希望比较这两个目录中的文件差异：

```sh
# 使用进程替换来比较两个目录的文件差异
diff <(ls foo) <(ls bar)
```

在这个示例中，`<(ls foo)` 和 `<(ls bar)` 分别会执行 `ls foo` 和 `ls bar` 命令，并将它们的输出放到临时文件中，然后 `diff` 命令比较这两个临时文件的内容。

### 详细示例

#### 命令替换示例

```sh
#!/bin/bash

# 获取当前日期和时间
current_date=$(date)
echo "当前日期和时间：$current_date"

# 获取系统的主机名
hostname=$(hostname)
echo "主机名：$hostname"
```

#### 进程替换示例

假设有两个文件 `file1.txt` 和 `file2.txt`，我们希望使用 `diff` 比较这两个文件的内容：

```sh
#!/bin/bash

# 创建示例文件
echo "This is file1" > file1.txt
echo "This is file2" > file2.txt

# 使用进程替换比较文件
diff <(cat file1.txt) <(cat file2.txt)
```

### 使用场景

1. **命令替换**：
   - 在脚本中获取命令输出并作为变量使用。
   - 在循环中迭代命令输出。
   
2. **进程替换**：
   - 当命令期望从文件而不是标准输入读取时使用。
   - 比较两个命令的输出，例如目录列表或文件内容。

Since that was a huge information dump, let’s see an example that showcases some of these features. It will iterate through the arguments we provide, `grep` for the string `foobar`, and append it to the file as a comment if it’s not found.

```sh
#!/bin/bash 

# 第一行是Shebang，告诉OS使用/bin/bash解释器来执行脚本。

echo "Starting program at $(date)" # Date will be substituted

echo "Running program $0 with $# arguments with pid $$"

for file in "$@"; do
    grep foobar "$file" > /dev/null 2> /dev/null
    #这行代码使用 grep 命令在指定的文件中搜索字符串 foobar，并将标准输出（STDOUT）和标准错误（STDERR）都重定向
    #到/dev/null，即丢弃输出。这意味着不管 grep 找到还是没有找到 foobar，输出都不会显示在终端上。
    
    # When pattern is not found, grep has exit status 1
    # We redirect STDOUT and STDERR to a null register since we do not care about them
    if [[ $? -ne 0 ]]; then
        echo "File $file does not have any foobar, adding one"
        echo "# foobar" >> "$file"
    fi
done
```

In the comparison we tested whether `$?` was not equal to 0. Bash implements many comparisons of this sort - you can find a detailed list in the manpage for [`test`](https://www.man7.org/linux/man-pages/man1/test.1.html). When performing comparisons in bash, try to use double brackets `[[ ]]` in favor of simple brackets `[ ]`. Chances of making mistakes are lower although it won’t be portable to `sh`. A more detailed explanation can be found [here](http://mywiki.wooledge.org/BashFAQ/031).

When launching scripts, you will often want to provide arguments that are similar. Bash has ways of making this easier, expanding expressions by carrying out filename expansion. These techniques are often referred to as shell *globbing*.

- Wildcards - Whenever you want to perform some sort of wildcard matching, you can use `?` and `*` to match one or any amount of characters respectively. For instance, given files `foo`, `foo1`, `foo2`, `foo10` and `bar`, the command `rm foo?` will delete `foo1` and `foo2` whereas `rm foo*` will delete all but `bar`.
- Curly braces `{}` - Whenever you have a common substring in a series of commands, you can use curly braces for bash to expand this automatically. This comes in very handy when moving or converting files.

```sh
convert image.{png,jpg}
# Will expand to
convert image.png image.jpg

cp /path/to/project/{foo,bar,baz}.sh /newpath
# Will expand to
cp /path/to/project/foo.sh /path/to/project/bar.sh /path/to/project/baz.sh /newpath

# Globbing techniques can also be combined
mv *{.py,.sh} folder
# Will move all *.py and *.sh files


mkdir foo bar
# This creates files foo/a, foo/b, ... foo/h, bar/a, bar/b, ... bar/h
touch {foo,bar}/{a..h}
touch foo/x bar/y
# Show differences between files in foo and bar
diff <(ls foo) <(ls bar)
# Outputs
# < x
# ---
# > y
```

Writing `bash` scripts can be tricky and unintuitive. There are tools like [shellcheck](https://github.com/koalaman/shellcheck) that will help you find errors in your sh/bash scripts.

Note that scripts need not necessarily be written in bash to be called from the terminal. For instance, here’s a simple Python script that outputs its arguments in reversed order:

```python
#!/usr/local/bin/python
import sys
for arg in reversed(sys.argv[1:]):
    print(arg)
```

The kernel knows to execute this script with a python interpreter instead of a shell command because we included a [shebang](https://en.wikipedia.org/wiki/Shebang_(Unix)) line at the top of the script. It is good practice to write shebang lines using the [`env`](https://www.man7.org/linux/man-pages/man1/env.1.html) command that will resolve to wherever the command lives in the system, increasing the portability of your scripts. 

Some differences between shell functions and scripts that you should keep in mind are:

- Functions have to be in the same language as the shell, while scripts can be written in any language. This is why including a shebang for scripts is important.
- Functions are loaded once when their definition is read. Scripts are loaded every time they are executed. This makes functions slightly faster to load, but whenever you change them you will have to reload their definition.
- Functions are executed in the current shell environment whereas scripts execute in their own process. Thus, functions can modify environment variables, e.g. change your current directory, whereas scripts can’t. Scripts will be passed by value environment variables that have been exported using [`export`](https://www.man7.org/linux/man-pages/man1/export.1p.html)
- As with any programming language, functions are a powerful construct to achieve modularity, code reuse, and clarity of shell code. Often shell scripts will include their own function definitions.

# Shell Tools

## Finding how to use commands

At this point, you might be wondering how to find the flags for the commands in the aliasing section such as `ls -l`, `mv -i` and `mkdir -p`. More generally, given a command, how do you go about finding out what it does and its different options? You could always start googling, but since UNIX predates StackOverflow, there are built-in ways of getting this information.

As we saw in the shell lecture, the first-order approach is to call said command with the `-h` or `--help` flags. A more detailed approach is to use the `man` command. Short for manual, [`man`](https://www.man7.org/linux/man-pages/man1/man.1.html) provides a manual page (called manpage) for a command you specify. For example, `man rm` will output the behavior of the `rm` command along with the flags that it takes, including the `-i` flag we showed earlier. In fact, what I have been linking so far for every command is the online version of the Linux manpages for the commands. Even non-native commands that you install will have manpage entries if the developer wrote them and included them as part of the installation process. For interactive tools such as the ones based on ncurses, help for the commands can often be accessed within the program using the `:help` command or typing `?`.

Sometimes manpages can provide overly detailed descriptions of the commands, making it hard to decipher what flags/syntax to use for common use cases. [TLDR pages](https://tldr.sh/) are a nifty complementary solution that focuses on giving example use cases of a command so you can quickly figure out which options to use. For instance, I find myself referring back to the tldr pages for [`tar`](https://tldr.inbrowser.app/pages/common/tar) and [`ffmpeg`](https://tldr.inbrowser.app/pages/common/ffmpeg) way more often than the manpages.

## Finding files

One of the most common repetitive tasks that every programmer faces is finding files or directories. All UNIX-like systems come packaged with [`find`](https://www.man7.org/linux/man-pages/man1/find.1.html), a great shell tool to find files. `find` will recursively search for files matching some criteria. Some examples:

```sh
# Find all directories named src
find . -name src -type d
# Find all python files that have a folder named test in their path
find . -path '*/test/*.py' -type f
# Find all files modified in the last day
find . -mtime -1
# Find all zip files with size in range 500k to 10M
find . -size +500k -size -10M -name '*.tar.gz'
```

Beyond listing files, find can also perform actions over files that match your query. This property can be incredibly helpful to simplify what could be fairly monotonous tasks.

```sh
# Delete all files with .tmp extension
find . -name '*.tmp' -exec rm {} \;
# Find all PNG files and convert them to JPG
find . -name '*.png' -exec convert {} {}.jpg \;
```

Despite `find`’s ubiquitousness, its syntax can sometimes be tricky to remember. For instance, to simply find files that match some pattern `PATTERN` you have to execute `find -name '*PATTERN*'` (or `-iname` if you want the pattern matching to be case insensitive). You could start building aliases for those scenarios, but part of the shell philosophy is that it is good to explore alternatives. Remember, one of the best properties of the shell is that you are just calling programs, so you can find (or even write yourself) replacements for some. For instance, [`fd`](https://github.com/sharkdp/fd) is a simple, fast, and user-friendly alternative to `find`. It offers some nice defaults like colorized output, default regex matching, and Unicode support. It also has, in my opinion, a more intuitive syntax. For example, the syntax to find a pattern `PATTERN` is `fd PATTERN`.

Most would agree that `find` and `fd` are good, but some of you might be wondering about the efficiency of looking for files every time versus compiling some sort of index or database for quickly searching. That is what [`locate`](https://www.man7.org/linux/man-pages/man1/locate.1.html) is for. `locate` uses a database that is updated using [`updatedb`](https://www.man7.org/linux/man-pages/man1/updatedb.1.html). In most systems, `updatedb` is updated daily via [`cron`](https://www.man7.org/linux/man-pages/man8/cron.8.html). Therefore one trade-off between the two is speed vs freshness. Moreover `find` and similar tools can also find files using attributes such as file size, modification time, or file permissions, while `locate` just uses the file name. A more in-depth comparison can be found [here](https://unix.stackexchange.com/questions/60205/locate-vs-find-usage-pros-and-cons-of-each-other).

## Finding code

Finding files by name is useful, but quite often you want to search based on file *content*. A common scenario is wanting to search for all files that contain some pattern, along with where in those files said pattern occurs. To achieve this, most UNIX-like systems provide [`grep`](https://www.man7.org/linux/man-pages/man1/grep.1.html), a generic tool for matching patterns from the input text. `grep` is an incredibly valuable shell tool that we will cover in greater detail during the data wrangling lecture.

For now, know that `grep` has many flags that make it a very versatile tool. Some I frequently use are `-C` for getting **C**ontext around the matching line and `-v` for in**v**erting the match, i.e. print all lines that do **not** match the pattern. For example, `grep -C 5` will print 5 lines before and after the match. When it comes to quickly searching through many files, you want to use `-R` since it will **R**ecursively go into directories and look for files for the matching string.

But `grep -R` can be improved in many ways, such as ignoring `.git` folders, using multi CPU support, &c. Many `grep` alternatives have been developed, including [ack](https://github.com/beyondgrep/ack3), [ag](https://github.com/ggreer/the_silver_searcher) and [rg](https://github.com/BurntSushi/ripgrep). All of them are fantastic and pretty much provide the same functionality. For now I am sticking with ripgrep (`rg`), given how fast and intuitive it is. Some examples:

```sh
# Find all python files where I used the requests library
rg -t py 'import requests'
# Find all files (including hidden files) without a shebang line
rg -u --files-without-match "^#\!"
# Find all matches of foo and print the following 5 lines
rg foo -A 5
# Print statistics of matches (# of matched lines and files )
rg --stats PATTERN
```

Note that as with `find`/`fd`, it is important that you know that these problems can be quickly solved using one of these tools, while the specific tools you use are not as important.

## Finding shell commands

So far we have seen how to find files and code, but as you start spending more time in the shell, you may want to find specific commands you typed at some point. The first thing to know is that typing the up arrow will give you back your last command, and if you keep pressing it you will slowly go through your shell history.

The `history` command will let you access your shell history programmatically. It will print your shell history to the standard output. If we want to search there we can pipe that output to `grep` and search for patterns. `history | grep find` will print commands that contain the substring “find”.

In most shells, you can make use of `Ctrl+R` to perform backwards search through your history. After pressing `Ctrl+R`, you can type a substring you want to match for commands in your history. As you keep pressing it, you will cycle through the matches in your history. This can also be enabled with the UP/DOWN arrows in [zsh](https://github.com/zsh-users/zsh-history-substring-search). A nice addition on top of `Ctrl+R` comes with using [fzf](https://github.com/junegunn/fzf/wiki/Configuring-shell-key-bindings#ctrl-r) bindings. `fzf` is a general-purpose fuzzy finder that can be used with many commands. Here it is used to fuzzily match through your history and present results in a convenient and visually pleasing manner.

Another cool history-related trick I really enjoy is **history-based autosuggestions**. First introduced by the [fish](https://fishshell.com/) shell, this feature dynamically autocompletes your current shell command with the most recent command that you typed that shares a common prefix with it. It can be enabled in [zsh](https://github.com/zsh-users/zsh-autosuggestions) and it is a great quality of life trick for your shell.

You can modify your shell’s history behavior, like preventing commands with a leading space from being included. This comes in handy when you are typing commands with passwords or other bits of sensitive information. To do this, add `HISTCONTROL=ignorespace` to your `.bashrc` or `setopt HIST_IGNORE_SPACE` to your `.zshrc`. If you make the mistake of not adding the leading space, you can always manually remove the entry by editing your `.bash_history` or `.zsh_history`.

## Directory Navigation

So far, we have assumed that you are already where you need to be to perform these actions. But how do you go about quickly navigating directories? There are many simple ways that you could do this, such as writing shell aliases or creating symlinks with [ln -s](https://www.man7.org/linux/man-pages/man1/ln.1.html), but the truth is that developers have figured out quite clever and sophisticated solutions by now.

As with the theme of this course, you often want to optimize for the common case. Finding frequent and/or recent files and directories can be done through tools like [`fasd`](https://github.com/clvv/fasd) and [`autojump`](https://github.com/wting/autojump). Fasd ranks files and directories by [*frecency*](https://web.archive.org/web/20210421120120/https://developer.mozilla.org/en-US/docs/Mozilla/Tech/Places/Frecency_algorithm), that is, by both *frequency* and *recency*. By default, `fasd` adds a `z` command that you can use to quickly `cd` using a substring of a *frecent* directory. For example, if you often go to `/home/user/files/cool_project` you can simply use `z cool` to jump there. Using autojump, this same change of directory could be accomplished using `j cool`.

More complex tools exist to quickly get an overview of a directory structure: [`tree`](https://linux.die.net/man/1/tree), [`broot`](https://github.com/Canop/broot) or even full fledged file managers like [`nnn`](https://github.com/jarun/nnn) or [`ranger`](https://github.com/ranger/ranger).
