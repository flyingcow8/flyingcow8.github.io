---
title: googletest和googlemock快速指南
date: 2017-10-31 16:53:00
categories: Develop
tags:
  - linux
  - googletest
  - googlemock
---

## x86平台编译捆绑版本(default)
```sh
cd googletest-release-1.8.0
mkdir mybuild
mkdir mylib
cd mybuild
cmake -DCMAKE_INSTALL_PREFIX=`pwd`/../mylib ..
make
make install
```
<!-- more -->

## 交叉编译
```sh
...
cmake -DCMAKE_INSTALL_PREFIX=`pwd`/../mylib -DCMAKE_TOOLCHAIN_FILE=`pwd`/../toolchain_hisiv300 ..
...
```

toolchain_hisiv300

```
set(CMAKE_SYSTEM_NAME Linux)
set(CMAKE_C_COMPILER /opt/hisi-linux/x86-arm/arm-hisiv300-linux/target/bin/arm-hisiv300-linux-gcc)
set(CMAKE_CXX_COMPILER /opt/hisi-linux/x86-arm/arm-hisiv300-linux/target/bin/arm-hisiv300-linux-g++)
```
> Builds a sample test. A test should link with either gtest.a or gtest_main.a, depending on whether it defines its own main() function.

googlemock和googletest分别会编译出两个静态库，g*.a和g*_main.a，两者的区别是g*_main.a中自带了main函数，用户不需要编写main函数了，只需要编写用例。本文使用g*.a这种类型的库。

## GOOGLEMOCK介绍
googlemock是一套写c++ mock类的框架（库），mock是指模拟的意思，主要在单元测试过程中模拟一些不易调用的、依赖生产环境的类，使整个测试能够进行下去。因此，googlemock单独使用并没有太大的意义，最好是配合单元测试框架一起工作。虽然它支持任何的C++测试框架，但和googletest配合起来最优（无缝集成）。默认编译的便是捆绑googletest版本的googlemock。
被mock的函数必需是属于某个类的虚函数或者纯虚函数，因此开发的时候就应该将那些比较依赖生产环境的函数定义成虚函数，比如数据库操作、网络IO等。

```c++
class Sample {
 public:
  Sample();
  virtual ~Sample();   //必须定义是虚函数
  int fun1();          //被测试的函数
  virtual int fun2();  //需要mock的函数
};

Sample::~Sample() {
}

int Sample::fun1() {
  if (fun2() == 0) 
    return 0;
  else
    return 1;
}

int Sample::fun2() {
  return connecting_server();
}
```
从被测试类继承一个MOCK类，调用框架中的宏来重新定义需要mock的函数。
在测试用例中写mock类函数的调用规则，比如返回值、调用次数等，调用mock类的对象进行单元测试。

```c++
#include "gmock/gmock.h"
#include "gtest/gtest.h"
#include "sample.h"
using ::testing::Return;
class MockSample : public Sample {
 public:
  MOCK_METHOD0(fun2, int());
};

TEST(AppTest, Sample) {
  MockSample sample;
  Sample ss;
  EXPECT_CALL(sample, fun2())
    .WillRepeatedly(Return(0));
  
  EXPECT_EQ(0, sample.fun1());
}

int main(int argc, char** argv) {
::testing::InitGoogleMock(&argc, argv);
return RUN_ALL_TESTS();
}
```

## GOOGLETEST介绍
google c++测试框架，被广泛使用，简单易用，输出界面友好。
使用TEST宏来写测试用例：

```c++
TEST(SampleFunTest, Negative) {
  EXPECT_EQ(1, SampleFun(-5));
  EXPECT_EQ(1, SampleFun(-1));
  EXPECT_GT(SampleFun(-10), 0);
}
```
TEST宏包含两个参数，分别是测试用例和测试项的名字。

```
ASSERT_* versions generate fatal failures when they fail, and abort the current function.
EXPECT_* versions generate nonfatal failures, which don't abort the current function.
```
一般情况下，使用EXPECT_*宏