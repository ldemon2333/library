# 1. this 指针
this作用域是在类内部，当在类的非静态成员函数中访问类的非静态成员的时候，编译器会自动将对象本身的地址作为一个隐含参数传递给函数。也就是说，即使你没有写上this指针，编译器在编译的时候也是加上this的，它作为非静态成员函数的隐含形参，对各成员的访问均通过this进行。

其次，this 指针的使用：
- 在类的非静态成员函数中返回类对象本身的时候，直接使用 `return *this`。
- 当参数与成员变量名相同时，如this->n = n （不能写成n = n)。

总结：this在成员函数的开始执行前构造，在成员的执行结束后清除。上述的get_age函数会被解析成`get_age(const A * const this)`,`add_age`函数会被解析成`add_age(A* const this,int a)`。在C++中类和结构是只有一个区别的：类的成员默认是private，而结构是public。this是类的指针，如果换成结构，那this就是结构的指针了。

在这段代码中，`&` 表示返回值是该对象的引用。

具体来说：

```cpp
Person& add_age(int a){
    age += a;
    return *this;
}
```

1. **`Person&`**: 表示函数 `add_age` 的返回类型是 `Person` 类型的引用。这意味着函数返回的是调用该函数的对象本身，而不是该对象的副本。
    
2. **`*this`**: `this` 是指向当前对象的指针，而 `*this` 则是解引用 `this` 指针，得到当前对象的引用。因此，`return *this;` 会返回当前对象本身。
    

返回对象的引用有一个重要的作用——**链式调用**。通过返回对象本身的引用，可以在一行中连续调用该函数，如下所示：

```cpp
Person p;
p.add_age(5).add_age(10).add_age(15);
```

这样，每次调用 `add_age` 函数时，都会修改对象的 `age` 并返回该对象自身，允许多个调用连接在一起。

如果没有 & 会触发 double free 问题：

在你的代码中出现 `double free` 的问题是因为你在 `add_age` 方法中返回了当前对象的副本，而 `Person` 类的析构函数会在该副本的生命周期结束时被调用，从而导致二次释放内存。具体来说：

### 问题分析：

1. **`add_age` 方法：**
    
    ```cpp
    Person add_age(int a){
       age += a;
       return *this;
    }
    ```
    
    在 `add_age` 方法中，`*this`（即当前对象的引用）被返回了一个副本。因为你返回的是一个 `Person` 对象的副本，而不是引用，这会导致以下两件事：
    
    - 在 `add_age` 返回时，编译器会复制当前对象（即创建一个新的 `Person` 对象）。
    - 新对象的析构函数会在它离开作用域时调用，从而销毁该副本并释放 `name` 指针。
2. **析构函数的问题：**
    
    ```cpp
    ~Person(){
       delete [] name;
    }
    ```
    
    `delete[] name;` 会释放对象的 `name` 指针的内存。在 `add_age` 中返回一个对象副本时，副本的析构函数会在离开作用域时被调用，导致副本的 `name` 被释放。
    
    然而，在 `main` 函数中的 `p.add_age(10)` 调用返回了副本，且 `p` 本身仍然存在，因此你会在程序结束时再调用一次析构函数，这时会试图再次释放 `name` 指针的内存，导致**双重释放（double free）**错误。
    

### 解释：

1. **返回引用**： `Person& add_age(int a)` 返回的是当前对象的引用（`*this`），而不是对象的副本。这样你就不会在返回时创建一个新副本，而是直接修改原始对象。
    
2. **避免双重释放**： 由于没有创建副本，`delete[] name` 只会在 `Person` 对象的生命周期结束时调用一次，不会出现双重释放的问题。

### 总结：

- 原来你在 `add_age` 中返回了一个对象的副本，导致该副本的析构函数在作用域结束时被调用，从而释放了 `name` 指针。之后，`p` 的析构函数再次释放 `name`，造成了双重释放。
- 通过修改 `add_age` 返回引用而不是副本，解决了双重释放的问题。