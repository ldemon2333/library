### 2.1.4 头文件名
对于纯粹的 C++ 头文件（iostream）来说，去掉 h 不只是形式上的变化，没有 h 的头文件也可以包含名称空间。

### 2.1.5 名称空间
如果使用iostream，而不是iostream.h，则应使用下面的名称空间编译指令来使iostream中的定义对程序可用：
```
using namespace std;
```

按照这种方式，类、函数和变量便是C++编译器的标准组件，它们现在都被放置在名称空间std中。仅当头文件没有扩展名h时，情况才是如此。这意味着在iostream中定义的用于输出的cout变量实际上是std::cout，而endl实际上是std::endl。因此，可以省略编译指令using，以下述方式进行编码：
![[Pasted image 20241209124652.png]]

这个using编译指令使得std名称空间中的所有名称都可用。这是一种偷懒的做法，在大型项目中一个潜在的问题。更好的方法是，只使所需的名称可用，这可以通过使用using声明来实现：
![[Pasted image 20241209125006.png]]

cout 是一个预定义的对象，知道如何显示字符串、数字和单个字符等。对象是类的特定实例，而类定义了数据的存储和使用方式。

cout 对象有一个接口，如果string 是一个字符串，可以显示该字符串：

然而，现在来看看C++从概念上如何解释这个过程。从概念上看，输出是一个流，即从程序流出的一系列字符。cout对象表示这种流，其属性是在iostream文件中定义的。cout的对象属性包括一个插入运算符（<<），它可以将其右侧的信息插入到流中。请看下面的语句（注意结尾的分号）：

![[Pasted image 20241209125404.png]]

它将字符串“Come up and C++ me some time.”插入到输出流中。
![[Pasted image 20241209125456.png]]

#### 1. 控制符endl
![[Pasted image 20241209125624.png]]
endl是一个特殊的C++符号。在输出流中插入endl将导致屏幕光标移到下一行开头。诸如endl等对于cout来说有特殊含义的特殊符号被称为控制符（manipulator）。和cout一样，endl也是在头文件iostream中定义的，且位于名称空间std中。

#### 2. 换行符
C++还提供了另一种在输出中指示换行的旧式方法：C语言符号\n：
![[Pasted image 20241209125750.png]]

本书中显示用引号括起的字符串时，通常使用换行符\n，在其他情况下则使用控制符endl。一个差别是，endl确保程序继续运行前刷新输出（将其立即显示在屏幕上）；而使用“\n”不能提供这样的保证，这意味着在有些系统中，有时可能在您输入信息后才会出现提示。

换行符是一种被称为“转义序列”的按键组合。

### 2.2.2 赋值语句
![[Pasted image 20241209130321.png]]

赋值将从右向左进行。首先，88被赋给steinway；然后，steinway的值（现在是88）被赋给baldwin；然后baldwin的值88被赋给yamaha（C++遵循C的爱好，允许外观奇怪的代码）。

cout，它是一个 ostream 类对象。ostream 类定义（iostream 文件的另一个成员）描述了 ostream 对象表示的数据以及可以对它执行的操作，如将数字或字符串插入到输出流中。类可以来自类库。ostream 和 istream 类就属于这种情况。它们并没有被内置到 C++ 语言中，而是语言标准指定的类。这些类定义位于 iostream 文件中，没有被内置到编译器中。如果愿意，程序员甚至可以修改这些类定义，虽然这不是一个好主意（准确地说，这个主意很糟）。iostream系列类和相关的fstream（或文件I/O）系列类是早期所有的实现都自带的唯一两组类定义。然而，ANSI/ISO C++委员会在C++标准中添加了其他一些类库。另外，多数实现都在软件包中提供了其他类定义。事实上，C++当前之所以如此有吸引力，很大程度上是由于存在大量支持UNIX、Macintosh和Windows编程的类库。

类描述指定了可对类对象执行的所有操作。要对特定对象执行这些允许的操作，需要给该对象发送一条消息。C++ 提供了两种发送消息的方式：
- 一种方式是使用类方法
- 重载运算符

### 2.4.2 函数变体
另外一些函数不接受任何参数。例如，有一个C库（与cstdlib或stdlib.h头文件相关的库）包含一个rand( )函数，该函数不接受任何参数，并返回一个随机整数。该函数的原型如下：
```
int rand(void);
```
关键字void明确指出，该函数不接受任何参数。如果省略void，让括号为空，则C++将其解释为一个不接受任何参数的隐式声明。

在有些语言中，有返回值的函数被称为函数（function）；没有返回值的函数被称为过程（procedure）或子程序（subroutine）。但C++与C一样，这两种变体都被称为函数。

# Chapter Review
![[Pasted image 20241209133440.png]]

1. Functions.
2. It causes the contents of the iostream file to be substituted for this directive before final compilation.
3. It makes definitions made in the std namespace available to a program.

