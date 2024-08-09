[Web](https://cs61a.org/articles/unix/)
### Removing Directories

We now know how to see, create, and move to directories. Our last command involving directories will be to delete them using the `rm` command.

First, let's create a exampleorary directory:

```
mkdir tmp
```

If you use the `ls` command, you should now see `tmp` listed as a directory.

Next, let's delete the `tmp` directory:

```
rm -r tmp
```

The `rm` command will **r**e**m**ove files and directories from your filesystem. By itself (that is, without the `-r`) the `rm` command only removes files. However, since we are removing a directory, we need to specify `-r` to **r**ecursively remove the `tmp` directory and any files that `tmp` might contain (the process is called "recursive" because, in order to remove `tmp`, we have to remove everything inside of `tmp`).

> As you've seen, some commands require arguments, like `mkdir`. Other commands do not require any arguments in order to work, like `ls`. In addition, most commands can also be given **flags**, like the `-r` for `rm`. Flags are ways to specify modified behavior for commands -- for example, `rm` by itself only removes files; using `-r` tells `rm` to remove directories.

### Renaming files

To rename files on Windows or MacOS, you would click on the name of the file and type in the new name.

Renaming files with the terminal can be a little confusing at first. Try the following command in the terminal (from the `example` directory)

```
mv unix.txt unix_commands.txt
```

Using `ls`, you'll see that `unix.txt` is gone -- in its place is a file called `unix_commands.txt`. Furthermore, typing `cat unix_commands.txt` will print out the same list of UNIX commands.

It appears that we renamed `unix.txt` to `unix_commands.txt` by using the `mv` command! Here's how to think about it:

- `mv` will move the contents of a file/directory into another file/directory. In the [previous section](https://cs61a.org/articles/unix/#moving-files), we moved a file into a directory.
- This time, we are moving the contents of a file (`unix.txt`) into another file (`unix_commands.txt`). While we are _technically_ moving file contents, this is _effectively_ the same thing as renaming a file!

This can be a bit confusing if you're seeing it for the first time, so make sure you understand it before you move onto the next section.

> **Note**: Suppose you already have two files, `alice.txt` and `bob.txt`, and you issue the command:
> 
> ```
> mv alice.txt bob.txt
> ```
> 
> This will overwrite the old contents of `bob.txt` with the contents of `alice.txt`! UNIX won't warn you about overwriting, so be careful when using the `mv` command!

### Copying files

Sometimes, it is useful to have multiple copies of a file. Try the following command:

```
cp unix_commands.txt new_file.txt
```

The `cp` command **c**o**p**ies the contents of one file into another file. Using `ls`, you will see that the `example` directory now contains two files, `unix_commands.txt` and `new_file.txt`. Using `cat` will verify that both files have the same contents.

Suppose we also wanted to copy the `unix_commands.txt` file to our home directory. Here's one way to do it:

1. Change [back to the home directory](https://cs61a.org/articles/unix/#moving-to-other-directories). (challenge: try doing this without looking up the command!)
2. Next, use the following command:
    
    ```
    cp example/unix_commands.txt .
    ```
    
    Don't forget the dot at the end!
    

The first argument (`example/unix_commands.txt`) tells the terminal to look in the `example` directory to find `unix_commands.txt`.

The second argument `.` tells the terminal to copy `unix_commands.txt` to the directory `.`. Just as two dots (`..`) represents the parent directory, a single dot (`.`) represents the _current_ directory (the directory we're in right now).

Now that we're in the home directory, we can use `ls` to verify that there is a copy of `unix_commands.txt`:

```
user@computer:~$ ls
Desktop/ ... unix_commands.txt ...
```

Using `cat unix_commands.txt` will show the same output of UNIX commands.

> **Recap**: we've seen two special directories: two dots `..` represents the parent directory (one directory up), while a single dot `.` represents the current directory. You can use these special expressions with any command that deals with directories. For example, you can `mv` a file to the current directory with the command
> 
> ```
> mv some_file .
> ```

