假设我们现在有`moduledemo`和`mypackage`两个包，其中`moduledemo`包中会导入`mypackage`包并使用它的`New`方法。

`mypackage/mypackage.go`内容如下：
```
package mypackage

import "fmt"

func New(){
	fmt.Println("mypackage.New")
}

```

分两种情况讨论：

# 在同一个项目下
**注意**：在一个项目（project）下我们是可以定义多个包（package）的。

现在的情况是，我们在`moduledemo/main.go`中调用了`mypackage`这个包。
```
moduledemo
├── go.mod
├── main.go
└── mypackage
    └── mypackage.go

```

### 导入包
这个时候，我们需要在`moduledemo/go.mod`中按如下定义：
```
module moduledemo

go 1.14
```

然后在`moduledemo/main.go`中按如下方式导入`mypackage`
```
package main

import (
	"fmt"
	"moduledemo/mypackage"  // 导入同一项目下的mypackage包
)
func main() {
	mypackage.New()
	fmt.Println("main")
}
```

## 举个例子
举一反三，假设我们现在有文件目录结构如下：

go.mod 与 main.go 平行，定义的模块代表当前路径，一个 go.mod 代表一个项目


# 不在同一个项目下
### 目录结构
```
├── moduledemo
│   ├── go.mod
│   └── main.go
└── mypackage
    ├── go.mod
    └── mypackage.go
```

### 导入包
这个时候，`mypackage`也需要进行module初始化，即拥有一个属于自己的`go.mod`文件，内容如下：
```
module mypackage

go 1.14
```

然后我们在`moduledemo/main.go`中按如下方式导入：
```
import (
	"fmt"
	"mypackage"
)
func main() {
	mypackage.New()
	fmt.Println("main")
}
```

因为这两个包不在同一个项目路径下，你想要导入本地包，并且这些包也没有发布到远程的github或其他代码仓库地址。这个时候我们就需要在`go.mod`文件中使用`replace`指令。

在调用方也就是`moduledemo/go.mod`中按如下方式指定使用相对路径来寻找`mypackage`这个包。
```
module moduledemo

go 1.14


require "mypackage" v0.0.0
replace "mypackage" => "../mypackage"
```


