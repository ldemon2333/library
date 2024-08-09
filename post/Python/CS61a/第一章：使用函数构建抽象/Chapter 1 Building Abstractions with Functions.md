# 1.1 Getting Started
The high productivity of computer science is only possible because the discipline is built upon an elegant and powerful set of fundamental ideas.

# 1.2 Elements of Programming
## 1.2.6 The Non-Pure Print Function
**Pure functions**
Functions have some input (their arguments) and return some output (the result of applying them). The built-in function
``` py
>>> abs(-2)
>>> 2
```

can be depicted as a small machine that takes input and produces output.

![](https://www.composingprograms.com/img/function_abs.png)

The function abs is _pure_. Pure functions have the property that applying them has no effects beyond returning a value. Moreover, a pure function must always return the same value when called twice with the same arguments.

**Non-pure functions.** In addition to returning a value, applying a non-pure function can generate _side effects_, which make some change to the state of the interpreter or computer. A common side effect is to generate additional output beyond the return value, using the print function.

Be careful with print! The fact that it returns None means that it _should not_ be the expression in an assignment statement.
``` py
>>> two = print(2)
>>> 2
>>> print(two)
>>> None
```

# 1.3 Defining New Functions
```py
def square(x):
	return mul(x,x)
```
which defines a new function that has been given the name *square*. This user-defined function is not built into the interpreter. It represents the compound operation of multiplying something by itself. The x in this definition is called a *formal parameter*, which provides a name for the thing to be multiplied.

## 1.3.1 Environments
What if a formal parameter has the same name as a built-in function? Can two functions share names without confusion? To resolve such questions, we must describe environments in more detail.

An environment in which an expression is evaluated consists of a sequence of _frames_, depicted as boxes. Each frame contains _bindings_, each of which associates a name with its corresponding value. There is a single _global_ frame. Assignment and import statements add entries to the first frame of the current environment. So far, our environment consists only of the global frame.

![[Pasted image 20240707163805.png]]

This _environment diagram_ shows the bindings of the current environment, along with the values to which names are bound.

Functions appear in environment diagrams as well. An import statement binds a name to a built-in function. A def statement binds a name to a user-defined function created by the definition. The resulting environment after importing mul and defining square appears below:

![[Pasted image 20240707163918.png]]

Each function is a line that starts with func, followed by the function name and formal parameters. Built-in functions such as mul do not have formal parameter names, and so ... is always used instead.

The name of a function is repeated twice, once in the frame and again as part of the function itself. The name appearing in the function is called the _intrinsic name_. The name in a frame is a _bound name_. There is a difference between the two: different names may refer to the same function, but that function itself has only one intrinsic name.

The name bound to a function in a frame is the one used during evaluation. The intrinsic name of a function does not play a role in evaluation. Step through the example below using the _Forward_ button to see that once the name max is bound to the value 3, it can no longer be used as a function.

![[Pasted image 20240707164216.png]]

The error message TypeError: 'int' object is not callable is reporting that the name max (currently bound to the number 3) is an integer and not a function. Therefore, it cannot be used as the operator in a call expression.

**Function Signatures.** Functions differ in the number of arguments that they are allowed to take. To track these requirements, we draw each function in a way that shows the function name and its formal parameters. The user-defined function square takes only x; providing more or fewer arguments will result in an error. A description of the formal parameters of a function is called the function's signature.

The function max can take an arbitrary number of arguments. It is rendered as max(...). Regardless of the number of arguments taken, all built-in functions will be rendered as <name>(...), because these primitive functions were never explicitly defined.

## 1.3.2 Calling User-Defined Functions
To evaluate a call expression whose operator names a user-defined function, the Python interpreter follows a computational process. As with any call expression, the interpreter evaluates the operator and operand expressions, and then applies the named function to the resulting arguments.

Applying a user-defined function introduces a second _local_ frame, which is only accessible to that function. To apply a user-defined function to some arguments:

1. Bind the arguments to the names of the function's formal parameters in a new _local_ frame.
2. Execute the body of the function in the environment that starts with this frame.

The environment in which the body is evaluated consists of two frames: first the local frame that contains formal parameter bindings, then the global frame that contains everything else. Each instance of a function application has its own independent local frame.

To illustrate an example in detail, several steps of the environment diagram for the same example are depicted below. After executing the first import statement, only the name mul is bound in the global frame.

![[Pasted image 20240707164723.png]]

First, the definition statement for the function square is executed. Notice that the entire def statement is processed in a single step. The body of a function is not executed until the function is called (not when it is defined).

Next, The square function is called with the argument -2, and so a new frame is created with the formal parameter x bound to the value -2.

Then, the name x is looked up in the current environment, which consists of the two frames shown. In both occurrences, x evaluates to -2, and so the square function returns 4.

The "Return value" in the square() frame is not a name binding; instead it indicates the value returned by the function call that created the frame.

Even in this simple example, two different environments are used. The top-level expression square(-2) is evaluated in the global environment, while the return expression mul(x, x) is evaluated in the environment created for by calling square. Both x and mul are bound in this environment, but in different frames.

The order of frames in an environment affects the value returned by looking up a name in an expression. We stated previously that a name is evaluated to the value associated with that name in the current environment. We can now be more precise:

**Name Evaluation.** A name evaluates to the value bound to that name in the earliest frame of the current environment in which that name is found.

Our conceptual framework of environments, names, and functions constitutes a _model of evaluation_; while some mechanical details are still unspecified (e.g., how a binding is implemented), our model does precisely and correctly describe how the interpreter evaluates call expressions. In Chapter 3 we will see how this model can serve as a blueprint for implementing a working interpreter for a programming language.

## 1.3.5 Choosing Names
The interchangeability of names does not imply that formal parameter names do not matter at all. On the contrary, well-chosen function and parameter names are essential for the human interpretability of function definitions!

The following guidelines are adapted from the [style guide for Python code](http://www.python.org/dev/peps/pep-0008), which serves as a guide for all (non-rebellious) Python programmers. A shared set of conventions smooths communication among members of a developer community. As a side effect of following these conventions, you will find that your code becomes more internally consistent.

1. Function names are lowercase, with words separated by underscores. Descriptive names are encouraged.
2. Function names typically evoke operations applied to arguments by the interpreter (e.g., print, add, square) or the name of the quantity that results (e.g., max, abs, sum).
3. Parameter names are lowercase, with words separated by underscores. Single-word names are preferred.
4. Parameter names should evoke the role of the parameter in the function, not just the kind of argument that is allowed.
5. Single letter parameter names are acceptable when their role is obvious, but avoid "l" (lowercase ell), "O" (capital oh), or "I" (capital i) to avoid confusion with numerals.

There are many exceptions to these guidelines, even in the Python standard library. Like the vocabulary of the English language, Python has inherited words from a variety of contributors, and the result is not always consistent.

# 1.4 Designing Functions

## 1.4.1 Documentation

A function definition will often include documentation describing the function, called a _docstring_, which must be indented along with the function body. Docstrings are conventionally triple quoted. The first line describes the job of the function in one line. The following lines can describe arguments and clarify the behavior of the function:
```py
>>> def pressure(v, t, n):
        """Compute the pressure in pascals of an ideal gas.

        Applies the ideal gas law: http://en.wikipedia.org/wiki/Ideal_gas_law

        v -- volume of gas, in cubic meters
        t -- absolute temperature in degrees kelvin
        n -- particles of gas
        """
        k = 1.38e-23  # Boltzmann's constant
        return n * k * t / v
```

When you call help with the name of a function as an argument, you see its docstring (type q to quit Python help).
```py
>>> help(pressure)
```

When writing Python programs, include docstrings for all but the simplest functions. Remember, code is written only once, but often read many times. The Python docs include [docstring guidelines](http://www.python.org/dev/peps/pep-0257/) that maintain consistency across different Python projects.

**Comments**. Comments in Python can be attached to the end of a line following the # symbol. For example, the comment Boltzmann's constant above describes k. These comments don't ever appear in Python's help, and they are ignored by the interpreter. They exist for humans alone.

# 1.5 控制
## 1.5.3 定义函数 II：局部赋值

最初，我们声明用户定义函数的主体仅由包含单个返回表达式的 `return` 语句组成。事实上，函数可以定义超出单个表达式的一系列操作。

每当用户定义的函数被调用时，其句体中的子句序列将会在局部环境中执行 --> 该环境通过调用函数创建的局部帧开始。`return` 语句会重定向控制：每当执行一个 `return` 语句时，函数应用程序就会终止，`return` 表达式的值会作为被调用函数的返回值。

赋值语句的作用是将名称与当前环境中的第一帧的值绑定。因此，函数体内的赋值语句不会影响全局帧。“函数只能操纵其局部帧”是创建模块化程序的关键，而在模块化程序中，纯函数仅通过它们接收和返回的值与外界交互。

# 1.6 高阶函数

我们对强大的编程语言提出的要求之一就是能够通过将名称分配给通用模板（general patterns）来构建抽象，然后直接使用该名称进行工作。函数提供了这种能力。正如我们将在下面的示例中看到的那样，代码中重复出现了一些常见的编程模板，但它们可以与许多不同的函数一起使用。这些模板也可以通过给它们命名来进行抽象。

为了将某些通用模板表达为具名概念（named concepts），我们需要构造一种“可以接收其他函数作为参数”或“可以把函数当作返回值”的函数。这种可以操作函数的函数就叫做高阶函数（higher-order functions）。本节将会展示：高阶函数可以作为一种强大的抽象机制，来极大地提高我们语言的表达能力。

## 1.6.1 作为参数的函数

这三个函数显然在背后共享着一个通用的模板（pattern）。它们在很大程度上是相同的，仅在名称和用于计算被加项 `k` 的函数上有所不同。我们可以通过在同一模板中填充槽位（slots）来生成每个函数：

```py
def <name>(n, <term>):
    total, k = 0, 1
    while k <= n:
        total, k = total + <term>(k), k + 1
    return total
```

这种通用模板的存在是一个强有力的证据 --> 表明了有一个实用的抽象手段正在“浮出水面”。这些函数都是用于求出各项的总和。作为程序设计者，我们希望我们的语言足够强大，以便我们可以编写一个表达“求和”概念的函数，而不仅仅是一个计算特定和的函数。在 Python 中，我们可以很轻易地做到这一点，方法就是使用上面展示的通用模板，并将“槽位”转换为形式参数：

## 1.6.6 柯里化

我们可以使用高阶函数将一个接受多个参数的函数转换为一个函数链，每个函数接受一个参数。更具体地说，给定一个函数 `f(x, y)`，我们可以定义另一个函数 `g` 使得 `g(x)(y)` 等价于 `f(x, y)`。在这里，`g` 是一个高阶函数，它接受单个参数 `x` 并返回另一个接受单个参数 `y` 的函数。这种转换称为柯里化（Curring）。

例如，我们可以定义 `pow` 函数的柯里化版本：
```py
>>> def curried_pow(x):
		def h(y):
			return pow(x,y)
		return h
>>> curried_pow(2)(3)
8
```

一些编程语言，例如 Haskell，只允许使用单个参数的函数，因此程序员必须对所有多参数过程进行柯里化。在 Python 等更通用的语言中，当我们需要一个只接受单个参数的函数时，柯里化就很有用。例如，map 模式（map pattern）就可以将单参数函数应用于一串值。
```py
>>> def map_to_range(start, end, f):
		while start < end:
			print(f(start))
			start +=1
```
我们可以使用 `map_to_range` 和 `curried_pow` 来计算 2 的前十次方，而不是专门编写一个函数来这样做：

```py
>>> map_to_range(0, 10, curried_pow(2))
1
2
4
8
16
32
64
128
256
512
```

我们可以类似地使用相同的两个函数来计算其他数字的幂。柯里化允许我们这样做，而无需为每个我们希望计算其幂的数字编写特定的函数。

在上面的例子中，我们手动对 `pow` 函数进行了柯里化变换，得到了 `curried_pow`。相反，我们可以定义函数来自动进行柯里化，以及逆柯里化变换（uncurrying transformation）：
```py
>>> def curry2(f):
		"""返回给定的双参数函数的柯里化的版本"""
		def g(x):
			def h(y):
				return f(x,y)
			return h
		return g
>>> def uncurry2(g):
		"""返回给定的柯里化函数的双参数版本"""
		def f(x, y):
			return g(x)(y)
		return f
>>> pow_curried = curry2(pow)
>>> pow_curried(2)(5)
32
>>> map_to_range(0, 10, pow_curried(2))
1 
2 
4 
8 
16 
32 
64 
128 
256 
512
```

`curry2` 函数接受一个双参数函数 `f` 并返回一个单参数函数 `g`。当 `g` 应用于参数 `x` 时，它返回一个单参数函数 `h`。当 `h` 应用于参数 `y` 时，它调用 `f(x, y)`。因此，`curry2(f)(x)(y)` 等价于 `f(x, y)` 。`uncurry2` 函数反转了柯里化变换，因此 `uncurry2(curry2(f))` 等价于 `f`。

```py
>>> uncurry2(pow_curried)(2, 5)
32
```

## 1.6.7 Lambda 表达式
```py
>>> def compose1(f, g):
        return lambda x: f(g(x))
```

lambda 表达式的结果称为 lambda 函数（匿名函数）。它没有固有名称（因此 Python 打印 `<lambda>` 作为名称），但除此之外它的行为与任何其他函数都相同。

```py
>>> s = lambda x: x * x
>>> s
<function <lambda> at 0xf3f490>
>>> s(12)
144
```

## 1.6.9 函数装饰器

Python 提供了一种特殊的语法来使用高阶函数作为执行 `def` 语句的一部分，称为装饰器（decorator）。最常见的例子也许就是 `trace` ：

```py
>>> def trace(fn):
        def wrapped(x):
            print('-> ', fn, '(', x, ')')
            return fn(x)
        return wrapped

>>> @trace
    def triple(x):
        return 3 * x

>>> triple(12)
->  <function triple at 0x102a39848> ( 12 )
36
```

在这个例子中，定义了一个高阶函数 `trace`，它返回一个函数，该函数在调用其参数前先输出一个打印语句来显示该参数。`triple` 的 `def` 语句有一个注解（annotation） `@trace`，它会影响 `def` 执行的规则。和往常一样，函数 `triple` 被创建了。但是，名称 triple 不会绑定到这个函数上。相反，这个名称会被绑定到在新定义的 `triple` 函数调用 `trace` 后返回的函数值上。代码中，这个装饰器等价于：

```py
>>> def triple(x):
        return 3 * x
>>> triple = trace(triple)
```

在本教材相关的项目中，装饰器被用于追踪，以及在从命令行运行程序时选择要调用哪些函数。

对于专家的额外内容：装饰器符号 `@` 也可以后跟一个调用表达式。跟在 `@` 后面的表达式会先被解析（就像上面的 'trace' 名称一样），然后是 `def` 语句，最后将装饰器表达式的运算结果应用到新定义的函数上，并将其结果绑定到 `def` 语句中的名称上。

例子：展示了如何使用带有参数的装饰器来记录函数的执行时间：
```py
import time

def timing_decorator(unit):
    def decorator(func):
        def wrapper(*args, **kwargs):
            start_time = time.time()
            result = func(*args, **kwargs)
            end_time = time.time()
            elapsed_time = end_time - start_time
            if unit == 'ms':
                elapsed_time *= 1000
                print(f"Execution time: {elapsed_time:.2f} ms")
            else:
                print(f"Execution time: {elapsed_time:.2f} s")
            return result
        return wrapper
    return decorator

@timing_decorator('ms')
def example_function():
    time.sleep(0.1)
    print("Function is running")

example_function()

```


