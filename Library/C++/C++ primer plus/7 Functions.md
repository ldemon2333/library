# Functions Arguments and Passing by Value
That means the numeric value of the argument is passed to the function, where it is assigned to a new variable.
![[Pasted image 20241212124046.png]]
Here `side` is a variable that, in the sample run, had the value 5. The function header for `cube()`, recall, was this:
![[Pasted image 20241212124127.png]]
When this function is called, it creates a new type `double` variable called `x` and initiallizes it with the value `5`. 这样，cube( )执行的操作将不会影响main( )中的数据，因为cube( )使用的是side的副本，而不是原来的数据。A variable that's used to receive passed valuses is called a *formal argument* or *formal parameter*. The value passed to the function is called the *actual argument*（实参）. The C++ Standard uses the word *argument* by itself to denote an actual argument or parameter and the word *parameter* by itself to denote a formal argument or parameter. Using this terminology, argument passing initializes the parameter to the argument.
![[Pasted image 20241212124924.png]]
Variables, including parameters, declared within a function are private to the function. When a function is called, the computer allocates the memory needed for these variables. When the function terminates, the computer frees the memory that was used for those variables. Such variables are called _local variables_ because they are localized to the function. As mentioned previously, this helps preserve data integrity. Such variables are also termed _automatic variables_.

## Multiple Arguments
As with single arguments, you don't have to use the same variable names in the prototype as in the definition, and you can omit the variable names in the prototype:
![[Pasted image 20241212125724.png]]
![[Pasted image 20241212125914.png]]

`cin.get()` functions read all input characters, including spaces and newlines, whereas `cin>>` skips spaces and newlines. 

# 7.3 函数和数组
The only new ingredient here is that you have to declare that one of the formal arguments is an array name. Let’s see what that and the rest of the function header look like:
![[Pasted image 20241212130354.png]]

`arr` is a pointer

## 7.3.1 函数如何使用指针来处理数组
C++将数组名解释为其第一个元素的地址：
![[Pasted image 20241212130528.png]]
First, the array declaration uses the array name to label the storages. Second, applying `sizeof` to an array name yields the size of the whole array, in bytes. Third, applying the address operator `&` to an array name returns the address of the whole array; for example, `&cookies` would be the address of a 32-byte block of memory if `int` is 4 bytes.

The correct function header should be this:
![[Pasted image 20241212131330.png]]
However, the array notation version (`int arr[]`) symbolically reminds you that `arr` not only points to an `int`, it points to the first `int` in an array of `int`s. 当指针指向数组的第一个元素时，本书使用数组表示法；而当指针指向一个独立的值时，使用指针表示法。别忘了，在其他的上下文中，int \*arr和int arr [ ]的含义并不相同。例如，不能在函数体中使用int tip[ ]来声明指针。
![[Pasted image 20241212131652.png]]

## 7.3.2 将数组作为参数意味着什么
这意味着，程序清单7.5实际上并没有将数组内容传递给函数，而是将数组的位置（地址）、包含的元素种类（类型）以及元素数目（n变量）提交给函数（参见图7.4）。有了这些信息后，函数便可以使用原来的数组。传递常规变量时，函数将使用该变量的拷贝；但传递数组时，函数将使用原来的数组。实际上，这种区别并不违反C++按值传递的方法，sum_arr( )函数仍传递了一个值，这个值被赋给一个新变量，但这个值是一个地址，而不是数组的内容
![[Pasted image 20241212131902.png]]
![[Pasted image 20241212132236.png]]![[Pasted image 20241212132243.png]]
First, note that `cookies` and `arr` both evaluate to the same address, exactly as claimed. But `sizeof cookies` is 32, whereas `sizeof arr` is only 4. That's because `sizeof cookies` is the size of the whole array, whereas `sizeof arr` is the size of the pointer variable. By the way, this is why you have to explicitly pass the size of the array rather than use `sizeof arr` in `sum_arr()`; the pointer by itself doesn't reveal the size of the array.

Because the only way `sum_arr()` knows the number of elements in the array is through whay you tell it with the second argument, you can lie to the function.
![[Pasted image 20241212132838.png]]