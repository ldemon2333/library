# Large files
In this assignment you'll increase the maximum size of an xv6 file. Currently xv6 files are limited to 286 blocks. This limit comes from the fact that an xv6 inode contains 12 "direct" block numbers and one "singly-indirect" block number, which refers to a block that holds up to 256 more block numbers, for a total of 12+256 = 268 blocks.

## Preliminaries
The `mkfs` program creates the xv6 file system disk image and determines how many total blocks the file system has; this size is control by `FSSIZE`

## What to Look at
`bmap()` is called both when reading and writing a file. When writing, `bmap()` allocates new blocks as needed to hold file content, as well as allocating an indirect block if needed to hold block addresses.

`bmap()` deals with two kinds of block numbers. The `bn` argument is a logical block number —— a block number within the file, relative to the start of the file. You can view `bmap()` as mapping a file's logical block numbers into disk block numbers.

# Symbolic links
Symbolic links (or soft links) refer to a linked file by pathname; when a symbolic link is opened, the kernel follows the link to the referred file. Symbolic links resembles hard links, but hard links are restricted to pointing to file on the same disk, while symbolic links can cross disk devices. 

Hints:
- Implement the `symlink(target, path)` system call to create a new symbolic link at path that refers to target. Note that target does not need to exist for the system call to succeed. You will need to choose somewhere to store the target path of a symbolic link, for example