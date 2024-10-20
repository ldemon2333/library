在Linux下，printf输出到控制台经历了app->libc->syscall->console驱动四个阶段

# 1.起于app
首先是用户态代码调用printf函数。printf函数首先会检查其格式字符串中的特殊字符，如%d、%s等，并根据这些特殊字符和传递给printf函数的参数来生成要输出的字符串。

# 2.经过libc
要能使用printf需要先包含：
```
#include<stdio.h>
```
因为printf它是一个libc实现的标准库函数，定义如下：
```
int printf(const char *format,...);
```
使用man 3 printf 查看手册：

总结下来就是“格式字符串的格式化”、“打印到标准输出”。

标准输出就是fd 1。

在Linux中，为了将格式化的输出发送到控制台上，printf函数会调用write系统调用。write调用会将生成的要输出的字符串写入到标准输出流中。而标准输出流（stdout）是一个文件描述符，它与控制台相关联。

首先要理解控制台的概念，控制台设备的实现方式可能因不同的Linux发行版而有所不同。例如，控制台是一个“/dev/pts/0”：
![[Pasted image 20241020132340.png]]
在嵌入式Linux设备上，控制台一般就是串口终端，设备端的驱动节点为“/dev/console":
![[Pasted image 20241020132419.png]]
另外，特别要注意的是标准输出它仅仅是一个描述符，它不等同于一个具体的设备也不会永远与之绑定，在某些应用场景下比如日志单独存储时可以被重定向，如下例子：
![[Pasted image 20241020132641.png]]

那么标准输出fd描述符究竟是如何与控制台相关联的呢？当write系统调用被调用时，数据又是怎么从用户空间下发到控制台的呢？

那无疑是先要看标准数据fd即控制台是什么时候被打开的。

子进程fork时默认继承父进程打开的文件描述符，又知道，在Linux的天下，用户进程兼为init进程的子民。

printf打印的内容本质上就是先打开一个控制台设备（嵌入式Linux下默认为/dev/console），这个描述符固定是标准输出（fd=1），然后往这个fd写内容。

# 3.syscall
当使用open系统调用打开/dev/console时，发生了什么？

当write系统调用被调用时，它会将数据从用户空间复制到内核空间。那么进一步又写到哪里去呢？

Linux下的文件分位设备文件、常规文件、管道、套接字等，不同的文件有它固有的写入方法。其中设备文件又分块设备文件和字符设备文件，而/dev/console就是后者：
![[Pasted image 20241020133648.png]]
其主设备号为5，次设备号为1.所以打开的时候，最终会找到当时注册的&console_cdev
![[Pasted image 20241020133821.png]]
也就是会打开console_fops的open方法即tty_open:
![[Pasted image 20241020133849.png]]
tty_open方法如下：
![[Pasted image 20241020133909.png]]

然后会调用tty_open_by_driver()->tty_lookup_driver()接口从注册的console驱动中查找一个最合适的tty对象（struct tty_struct * tty），最终会调用tty_and_file()将tty对象绑定到打开的file对象中：
![[Pasted image 20241020134112.png]]
小结下就是open的最终目标其实找到tty对象绑定到file对象，为接下来写做铺垫。

在Linux系统下，write系统调用是sys_write():
![[Pasted image 20241020134222.png]]
上面可以看到sys_write()会进一步调用vrf_write():
![[Pasted image 20241020134300.png]]
所以，write的时候也自然会调用console_fops的write方法即ssize_t redirected_tty_write()，正常就会继续调用tty_write():
![[Pasted image 20241020134422.png]]

# 4. 穿越tty层和uart层
tty它是一个框架，为了实现分层与隔离，向下兼容不同的硬件，对上提供统一的终端操作接口。
![[Pasted image 20241020134623.png]]
前面讲到了tty_write()，所处的层次是tty核心层，会将数据从用户态拷贝到tty缓冲区。向下还有行规层，其作用是用来控制终端的解释行为，比如要不要换行转换、回显等。行程层的操作方法是n_tty_ops:

会继续调用n_tty_write()将数据发到tty驱动层，其操作方法转换了struct tty_struct * tty 对象的ops->write()方法。