---
title: "C和C++函数相互调用"
date: 2018-03-25 14:30:00
categories: Programing
tags:
  - C/C++
---
> C和C++分别有各自大量优秀的开源库，C++工程需要调用到C库或者C工程需要调用C++库的情况比较常见。虽然C和C++有天然近亲关系，但函数互相调用还是需要有一些特殊处理。
<!-- more -->
## 链接指示：extern "C"
C++程序使用链接指示（linkage directive）指出任意非C++函数所用的语言。extern是关键字，"C"指代应该根据C的编译和链接规则来链接。因此extern "C"只能在c++编译环境下使用。如果C文件中加入该链接指示语句，则编译时会出错。一般情况下会在头文件声明函数时使用__cplusplus宏来区分是否使用extern "C"，比如：

```c++
#ifdef _cplusplus
extern "C" {
#endif

#include <string.h>

void func();

#ifdef _cplusplus
}
#endif
```
__cplusplus宏是由c++编译器预定义的，无需自己定义，c编译器并不会定义该宏。链接指示后面跟上花括号括起来可以同时声明若干函数，也可以不加花括号，单独声明一条函数。

当一个#include指示被放置在复合链接指示的花括号中时，头文件中的所有普通函数声明都被认为是由链接指示的语言编写的。并且链接指示可以嵌套，因此如果头文件包含带自带链接指示的函数，则该函数的链接不受影响。

我们使用链接指示extern "C"绝大部分的情况是对函数声明，它对作为返回类型或形参类型的函数指针也是有效的：

```c++
//pf指向一个C函数，该函数接受一个int返回void
extern "C" void (*pf)(int);

// f1是一个C函数，并且，它的形参是一个指向C函数的指针
extern "C" void f1(void (*)(int));
```

## C++调用C
C++作为一种面向对象的编程语言，特点之一是支持重载。函数重载是指在同一作用域内，可以有一组具有相同函数名，不同参数列表的函数，这组函数被称为重载函数。
如果在一个c源文件中定义了一个函数：
> void hello()

然后我们将编译得到的a.out反汇编后输出：$objdump -d a.out > a.txt，我们找到这个函数编译后的名字是
> hello

如果在一个c++源文件中定义了三个重载函数：
> void hello()

> void hello(int i) 

> void hello(int i, char *s)

这三个函数编译后的名字分别是：
> _Z5hellov

> _Z5helloi

> _Z5helloiPc

我们发现C++中函数编译后的名字已经变成带返回值和参数的形式了。因此如果hello()函数是在C代码中编译得到的，在C++中直接调用该函数，C++会去找_Z5hellov这个函数名，而实际的函数名是hello，链接时当然会出现函数未定义的错误。
	
因些C++中调用C函数，在函数声明的时候需要加上链接指示extern "C"，使得链接时C++链接器去找名字叫hello的函数而不是_Z5hellov的函数。我们可以在C++中调用C函数前手动声明一下，比如：

```c++
// main.cc
#include <iostream>

extern "C" void hello();

int main() {
  hello();
  std::cout << "it is a sample" << std::endl;
  return 0;
}
```
当然，更好的做法是在C代码或者C库的头文件中来指示：

```c++
#ifndef HELLO_H
#def HELLO_H

#ifdef _cplusplus
extern "C" {
#endif

void hello();

#ifdef _cplusplus
}
#endif
#endif //HELLO_H
```
这样的话，C++源文件只需要加入这个头文件即可。而且因为使用了__cplusplus，这个头文件在c和c++中使用均能被正确地编译。

## C调用C++
由于C++语言向下兼容C语言，因此C++调用C比较简单，一个extern "C"即可搞定。但是C程序却根本无法理解类、命名空间、构造函数、析构函数、重载等这些C++特有的类型和操作，因此C调用C++的情况稍显麻烦一些。

简单来说，我们可以在C++函数定义的时候直接加上链接指示，比如：

```c++
//hello.cc
#include <iostream>

extern "C" void hello(const char *name) 
{
  std::cout << "hello " << name << std::endl;
}
```
C++编译器就能将该函数生成适合于C程序的代码，我们可以在C文件中声明并调用该函数。

```c++
//main.c
extern void hello(const char *name);

int main()
{
  const char *s = "Roger";
  hello(s);
  return 0;
}
```
另外一种做法是在CC文件中用链接指示重新声明一下需要被C调用的函数，这样在函数定义处就不需要加链接指示了。

```c++
//hello.cc
//hello.cc
#include <iostream>
extern "C" void hello(const char *name);

void hello(const char *name) 
{
  std::cout << "hello " << name << std::endl;
}
```
当然，更合理的做法是在头文件中用__cplusplus和extern "C"来声明函数。同时在调用这些函数的C文件中包含该头文件，更为关键的是在定义这些函数的CC文件中必须包含该头文件。

extern "C"来定义C++中的函数会使其丧失重载性，例如：

```c++
//hello.cc
#include <iostream>
extern "C" void hello(const char *name);
extern "C" void hello();

void hello(const char *name) 
{
  std::cout << "hello " << name << std::endl;
}

void hello()
{
  std::cout << "hello " << std::endl;

}
```
这样，编译的时候会报重复定义的错误。

实际状况并非如此简单，比如调用C++库的情况下，我们无法改动C++源码。因此多数情况下我们需要编写一个中间接口(包裹函数)来间接调用到C++函数，并且这个中间接口必须处于一个CC/CPP文件中。以一个工程实际问题为例，glog是一款优秀的日志工具，但它是C++编写的，比如：

```c++
//glog初始化
google::InitGoogleLogging("my_app");

//输出LOG消息
LOG(WARNING) << "it is a log msg!";
```
google是一个命名空间，<< 是C++输出风格，C编译器根本无法理解这些语句。我们新建一个cc和h文件来写包裹函数。

```c++
//log_wrapper.cc
#include <glog/logging.h>
#include "log_wrapper.h"

void log_init()
{
  google::InitGoogleLogging("my_app");
}

void log_warn(const char *s)
{
 LOG(WARNING) << s;
}
```

```c++
//log_wrapper.h
#ifndef HELLO_H
#def HELLO_H

#ifdef _cplusplus
extern "C" {
#endif

void log_init();

void log_warn(const char *s);

#ifdef _cplusplus
}
#endif
#endif //HELLO_H
```

```c++
//main.c
#include "log_wrapper.h"

int main()
{
 log_init();
 log_warn("it is a warning msg");
 return 0;
```
至于如何构建，以gcc/g++为例：

```
CFLAGS=-g -Wall -I./include

objects = main.o log_wrapper.o

sample: $(objects)
	gcc $(CFLAGS) -o sample $(objects) -pthread -L ./ -lglog -lstdc++

main.o: main.c
	gcc $(CFLAGS) -c main.c 

log_wrapper.o: log_wrapper.cc
	g++ $(CFLAGS) -c log_wrapper.cc

clean:
	rm -f sample $(objects)
```
当然，也可以用g++来链接，相应语句替换为:

```
g++ $(CFLAGS) -o sample $(objects) -pthread -L ./ -lglog
```

## 总结
1. extern "C"只能在C++中使用。
2. C++调用C函数直接用extern "C"指示一下函数声明即可。
3. C++的函数如果需要被C调用，需要在定义的文件中用extern "C"来声明。
4. 从通用性考虑，C调用C++函数需要写包裹函数
