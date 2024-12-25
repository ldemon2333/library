### 3.1.4 无符号类型
unsigned 本身是 unsigned int 的缩写。

### Interger Literals
![[Pasted image 20241209134858.png]]
By default, cout displays integers in decimal form, regardless of how they are written in a program, as the following output shows:
![[Pasted image 20241209134948.png]]

By the way, if you want to display a value in hexadecimal or octal form, you can use some special features of `cout`. Recall that the iostream header file provides the `endl` manipulator to give cout the message to start a new line. Similarly, it provides the `dec`, `hex`, and `oct` manipulators to give cout the messages to display integers in decimal, hexadecimal, and octal formats, respectively. Listing 3.4 uses hex and oct to display the decimal value 42 in three formats. (Decimal is the default format, and each format stays in effect
until you change it.)

![[Pasted image 20241209135332.png]]
因此，控制符hex实际上是一条消息，告诉cout采取何种行为。另外，由于标识符hex位于名称空间std中，而程序使用了该名称空间，因此不能将hex用作变量名。然而，如果省略编译指令using，而使用std::cout、std::endl、std::hex和std::oct，则可以将hex用作变量名。

### 3.1.7 C++ 如何确定常量的类型
First, look at the suffixes. These are letters placed at the end of a numeric constant to indicate the type. An `l` or `L` suffix on an integer means the integer is a type `long` constant, a `u` or `U` suffix indicates an `unsigned int` constant, and `ul` (in any combination of orders and uppercase and lowercase) indicates a type `unsigned long` constant. For example, on a system using a 16-bit `int` and a 32-bit `long`, the number 22022 is stored in 16 bits as an `int`, and the number `22022L` is stored in 32 bits as a `long`. Similarly, `22022LU` and `22022UL` are `unsigned long`. C++11 provides the `ll` and `LL` suffixes for type `long long`, and `ull`, `Ull`, `uLL` for `unsigned long long`.

在 C++ 中，对十进制整数采用的规则，与十六进制和八进制不同。对于不带后缀的十进制整数，将使用下面几种类型中能够存储该数的最小类型来表示：int、long或long long。在int为16位、long为32位的计算机系统上，20000被表示为int类型，40000被表示为long类型，3000000000被表示为long long类型。对于不带后缀的
十六进制或八进制整数，将使用下面几种类型中能够存储该数的最小类型来表示：int、unsigned int long、unsigned long、long long或unsigned long long。在将40000表示为long的计算机系统中，十六进制数0x9C40（40000）将被表示为unsigned int。这是因为十六进制常用来表示内存地址，而内存地址是没有符号的，因此，usigned int比long更适合用来表示16位的地址。

the `cout.put()` function, which displays a single character.

## A Member Function: cout.put()
The `cout.put()` is *member function*. A class defines how to represent data and how to manipulate it. A member function belongs to a class and describes a method for manipulating class data. You use a period to combine the object name (`cout`) with the function name (`put()`). The period is called the *membership operator*.

# 3.2 const Qualifier
初始化后就是只读变量

const 与 \#define 
For one thing, it lets you specify the type explicitly. Second, you can use C++'s scoping rules to limit the definition to particular functions or files. Third, you can use `const` with more elaborate types, such as arrays and structures.

### 3.4.5 auto Declarations in C++11
自动推断变量的 type

# 3.5 总结
整型从最小到最大依次是：bool, char, signed char, unsigned char, short, unsigned short, int, unsigned int, long, unsigned long 以及C++11 新增的 long long 和 unsigned long long.

浮点类型：float, double, long double.


