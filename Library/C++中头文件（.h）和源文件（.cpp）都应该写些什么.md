# 头文件(.h)
写类的声明（包括类里面的成员和方法的声明）、函数原型、#define常数等，但一般来说不写出具体的实现。

在写头文件时需要注意，在开头和结尾处必须按照如下样式加上预编译语句（如下）：
```
#ifndef CIRCLE_H
#define CIRCLE_H
// code
#endif
```
这样做是为了防止重复编译，不这样做就有可能出错。

至于CIRCLE_H这个名字实际上是无所谓的，你叫什么都行，只要符合规范都行。原则上来说，非常建议把它写成这种形式，因为比较容易和头文件的名字对应。

# 源文件(.cpp)
源文件主要写实现头文件中已经声明的那些函数的具体代码。需要注意的是，开头必须#include一下实现的头文件，以及要用到的头文件。**那么当你需要用到自己写的头文件中的类时，只需要#include进来就行了

下面举个最简单的例子来描述一下，咱就求个圆面积

- 在头文件的文件夹里新建一个名为 Circle.h 的头文件
```
#ifndef CIRCLE_H
#define CIRCLE_H
class Circle{
	private:
		double r;
	public:
		Circle();//构造函数
		Circle(double R);//构造函数
		double Area();
};
#endif
```

注意到开头结尾的预编译语句。在头文件里，并不写出函数的具体实现。
- 第3步，要给出Circle类的具体实现，因此，在源文件夹里新建一个Circle.cpp的文件，它的内容如下：
```C++
#include"Circle.h"
Circle::Circle(){
	this->r = 5.0;
}
Circle::Circle(double R){
	this->r = R;
}
double Circle::Area(){
	return 3.14*r*r;
}
```
- 最后，我们建一个 main.cpp 来测试我们写的Circle类，内容如下：
```
#include<iostream>
#include"Circle.h"
using namespace std;
int main(){
	Circle c(3);
	cout<<"Area="<<c.Area()<<endl;
	return 1;
}
```
