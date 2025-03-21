![[Pasted image 20250217165602.png]]
可以使用`-h`显示选项帮助信息：
![[Pasted image 20250217165615.png]]
总结一下，使用`flag`库的一般步骤：

- 定义一些全局变量存储选项的值，如这里的`intflag/boolflag/stringflag`；
- 在`init`方法中使用`flag.TypeVar`方法定义选项，这里的`Type`可以为基本类型`Int/Uint/Float64/Bool`，还可以是时间间隔`time.Duration`。定义时传入变量的地址、选项名、默认值和帮助信息；
- 在`main`方法中调用`flag.Parse`从`os.Args[1:]`中解析选项。因为`os.Args[0]`为可执行程序路径，会被剔除。

# 选项格式
遇到第一个非选项参数（即不是以`-`和`--`开头的）或终止符`--`，解析停止。运行下面程序：
![[Pasted image 20250217170357.png]]

解析终止之后如果还有命令行参数，`flag`库会存储下来，通过`flag.Args`方法返回这些参数的切片。 可以通过`flag.NArg`方法获取未解析的参数数量，`flag.Arg(i)`访问位置`i`（从 0 开始）上的参数。 选项个数也可以通过调用`flag.NFlag`方法获取。

# 另一种定义选项的方式
上面我们介绍了使用`flag.TypeVar`定义选项，这种方式需要我们先定义变量，然后变量的地址。 还有一种方式，调用`flag.Type`（其中`Type`可以为`Int/Uint/Bool/Float64/String/Duration`等）会自动为我们分配变量，返回该变量的地址。用法与前一种方式类似：
![[Pasted image 20250217170643.png]]
除了使用时需要解引用，其它与前一种方式基本相同。
