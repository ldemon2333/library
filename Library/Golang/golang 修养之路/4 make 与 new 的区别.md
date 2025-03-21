变量的声明我们可以通过var关键字，然后就可以在程序中使用。当我们不指定变量的默认值时，这些变量的默认值是他们的零值，比如int类型的零值是0,string类型的零值是`""`，引用类型的零值是`nil`。

```
package main

import (
 "fmt"
)

func main() {
   var i *int
   *i=10
   fmt.Println(*i)
}
```
```
$ go run test1.go 
panic: runtime error: invalid memory address or nil pointer dereference
[signal SIGSEGV: segmentation violation code=0x1 addr=0x0 pc=0x4849df]

goroutine 1 [running]:
main.main()
	/home/itheima/go/src/golang_deeper/make_new/t
```

从这个提示中可以看出，对于引用类型的变量，我们不光要声明它，还要为它分配内容空间，否则我们的值放在哪里去呢？这就是上面错误提示的原因

# new
​ 它只接受一个参数，这个参数是一个类型，分配好内存后，返回一个指向该类型内存地址的指针。同时请注意它同时把分配的内存置为零，也就是类型的零值
```
package main

import (
    "fmt"
    "sync"
)

type user struct {
    lock sync.Mutex
    name string
    age int
}

func main() {

    u := new(user) //默认给u分配到内存全部为0

    u.lock.Lock()  //可以直接使用，因为lock为0,是开锁状态
    u.name = "张三"
    u.lock.Unlock()

    fmt.Println(u)
}
```

示例中的user类型中的lock字段我不用初始化，直接可以拿来用，不会有无效内存引用异常，因为它已经被零值了

这就是new，它返回的永远是类型的指针，指向分配类型的内存地址

# make
make也是用于内存分配的，但是和new不同。

它只用于
- chan
- map
- slice
的内存创建，而且它返回的类型就是这三个类型本身，而不是他们的指针类型，因为这三种类型就是引用类型，所以就没有必要返回他们的指针了。

注意，因为这三种类型是引用类型，所以必须得初始化，但是不是置为零值，这个和new是不一样的。

# 异同
相同

- 堆空间分配

不同

make: 只用于slice、map以及channel的初始化， 无可替代

new: 用于类型内存分配(初始化值为0)， 不常用

> new不常用
> 
> 所以有new这个内置函数，可以给我们分配一块内存让我们使用，但是现实的编码中，它是不常用的。我们通常都是采用短语句声明以及结构体的字面量达到我们的目的，比如：
> 
> ```go
> i : =0
> u := user{}
> ```

> make 无可替代
> 
> 我们在使用slice、map以及channel的时候，还是要使用make进行初始化，然后才才可以对他们进行操作。

