# plan9 汇编
还有一点区别是在使用 MOVQ 的时候会看到带括号和不带括号的区别。
```
// 加括号代表是指针的引用
MOVQ (AX), BX   // => BX = *AX 将AX指向的内存区域8byte赋值给BX
MOVQ 16(AX), BX // => BX = *(AX + 16)

// 不加括号是值的引用
MOVQ AX, BX     // => BX = AX 将AX中存储的内容赋值给BX，注意区别
```

地址运算：
```
LEAQ (AX)(AX*2), CX // => CX = AX + (AX * 2) = AX * 3
```

![[Pasted image 20250402101306.png]]stack frame size：包含局部变量以及额外调用函数的参数空间；

arguments size：包含参数以及返回值大小，例如入参是 3 个 int64 类型，返回值是 1 个 int64 类型，那么返回值就是 sizeof(int64) * 4；

# 闭包
a **closure** is a record storing **a function** together with **an environment**.

**闭包**是由**函数**和与其相关的引用**环境**组合而成的实体
```go
package main

func test() func() {
    x := 100
    return func() {
        x += 100
    }
}

func main() {
    f := test()
    f() //x= 200
    f() //x= 300
    f() //x= 400
} 
```
由于闭包是有上下文的，我们以测试例子为例，每调用一次 f() 函数，变量 x 都会发生变化。但是我们通过其他的方法调用都知道，如果变量保存在栈上那么变量会随栈帧的退出而失效，所以闭包的变量会逃逸到堆上。

我们可以进行逃逸分析进行证明：

```sh
[root@localhost gotest]$ go run -gcflags "-m -l" main.go 
# command-line-arguments
./main.go:4:2: moved to heap: x
./main.go:5:9: func literal escapes to heap
```

可以看到变量 x 逃逸到了堆中。


#### 小结

通过上面的分析，可以发现其实匿名函数就是闭包的一种，只是没有传递变量信息而已。而在闭包的调用中，会将上下文信息逃逸到堆上，避免因为栈帧调用结束而被回收。

在上面的例子闭包函数 test 的调用中，非常复杂的做了很多变量的传递，其实就是做了这几件事：

1. 为上下文信息初始化内存块；
2. 将上下文信息的地址值保存到 AX 寄存器中；
3. 将闭包函数封装好的 test.func1 调用函数地址写入到 caller 的栈顶；

这里的上下文信息指的是 x 变量以及 test.func1 函数。将这两个信息地址写入到 AX 寄存器之后回到 main 函数，获取到栈顶的函数地址写入到 AX 执行 `CALL AX` 进行调用。

因为 x 变量地址是写入到 AX + 8 的位置上，所以在调用 test.func1 函数的时候是通过获取 AX + 8 的位置上的值从而获取到 x 变量地址从而做到改变闭包上下文信息的目的。

