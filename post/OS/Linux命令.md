# wget命令

`wget` 全称 web get，命令行中下载文件的工具。可以跟踪HTML页面上的链接依次下载来创建远程服务器的本地版本，完全重建原始站点的目录结构，这又称作“递归下载”。

**1. 命令格式**

`wget` [参数] [URL地址]

**2.命令参数：**

 + 递归下载参数：

   -r，--recursive	递归下载 

 + Recursive accept/reject：

	-np，--no-parent	don't ascend to the parent directory

+ Directories:

​		-nH，--no-host-directories	don't create host directories

---



# rm命令

### Using the `rm` Command

The `rm` (remove) command is the most commonly used command to delete files in Linux.

1. **Delete a single file**:

	```sh
   rm filename.txt
   ```

   - This command deletes `filename.txt`.

2. **Delete multiple files**:

   ```sh
   rm file1.txt file2.txt file3.txt
   ```

   - This command deletes `file1.txt`, `file2.txt`, and `file3.txt`.

3. **Delete a file without prompting for confirmation**:

   ```sh
   rm -f filename.txt
   ```

   - The `-f` (force) option forces the deletion without prompting for confirmation.

---

# 新建一个文件

### Using Command-Line Tools

1. **Using `touch`**:

   - `touch` creates an empty file if the file does not already exist.

   ```sh
   touch filename.txt
   ```

   - This command will create an empty text file named `filename.txt`.

2. **Using `echo`**:

   - `echo` can be used to create a file with some initial content.

   ```sh
   echo "Initial content" > filename.txt
   ```

   - This command creates a file named `filename.txt` with the content "Initial content".

3. **Using `cat`**:

   - `cat` can be used to create a file and input multiple lines of text.

   ```sh
   cat > filename.txt
   ```

   - After running the command, type the text you want to include in the file. Press `Ctrl+D` to save and exit.  

     

   **In the command `echo "Initial content" > filename.txt`, the `>` symbol is used for output redirection in the shell. Here's a detailed explanation:**内容重定向

   ### Explanation of the Components

   1. **`echo "Initial content"`**:
      - The `echo` command is used to display a line of text or a string. In this case, it outputs the string `"Initial content"`.

   2. **`>` (Redirection Operator)**:
      - The `>` symbol is a redirection operator that directs the output of a command to a file instead of the terminal.
      - When used, it creates a new file with the specified name if it does not already exist, or it truncates the file (empties its contents) if it already exists, before writing the output to the file.

   3. **`filename.txt`**:
      - This is the name of the file to which the output of the `echo` command is redirected.

   ### What Happens in the Command

   ```sh
   echo "Initial content" > filename.txt
   ```

   - The shell executes the `echo` command, which produces the output `Initial content`.
   - The `>` operator takes this output and writes it to `filename.txt`.
   - If `filename.txt` does not exist, it is created.
   - If `filename.txt` already exists, its current contents are overwritten with `Initial content`.

   ### Example Scenario

   1. **File Creation**:
      - If `filename.txt` does not exist, running the command will create the file and write `Initial content` to it.
      ```sh
      $ echo "Initial content" > filename.txt
      $ cat filename.txt
      Initial content
      ```

   2. **Overwriting Existing File**:
      - If `filename.txt` already exists, running the command will overwrite its contents with `Initial content`.
      ```sh
      $ echo "Some old content" > filename.txt
      $ echo "Initial content" > filename.txt
      $ cat filename.txt
      Initial content
      ```

   ### Appending to a File

   If you want to append to a file instead of overwriting it, you can use the `>>` operator:

   ```sh
   echo "Additional content" >> filename.txt
   ```

   - This command appends `Additional content` to `filename.txt` without removing the existing content.
   ```sh
   $ echo "Initial content" > filename.txt
   $ echo "Additional content" >> filename.txt
   $ cat filename.txt
   Initial content
   Additional content
   ```

   ### Summary

   - `>`: Redirects output to a file, creating or overwriting the file.
   - `>>`: Redirects output to a file, appending to the file if it exists.

   Using these redirection operators allows you to control where the output of your commands is sent, which is useful for creating and modifying files directly from the command line.

---

# pwd命令

显示当前工作路径

