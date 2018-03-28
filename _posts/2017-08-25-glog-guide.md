---
title: glog使用指南
date: 2017-08-25 09:57:00
categories: Develop
tags:
  - linux
  - glog
---
glog是Google的开源C++日志框架，简单易用，示例应用代码如下：

```c++
#include <glog/logging.h>
int main(int argc, char* argv[]) {     // Initialize Google's logging library.     google::InitGoogleLogging(argv[0]);

     LOG(INFO) << "it is a log msg";
}
```
<!-- more -->

## 严重性等级
glog定义了四种严重性等级，从低到高依次为: INFO, WARNING, ERROR和FATAL，对应的数字编号为0、1、2、3。FATAL类型的消息会终止整个程序，一般情况下可以使用DFATAL代替FATAL。DFATAL在DEBUG模式下等同于FATAL，而在Release模式下（生产环境）自动降低严重性等级为ERROR，从而保证应用程序不被终止。区分RELEASE和DEBUG模式的方式是是否定义NDEBUG宏（构建RELEASE版本时需要加入-DNDEBUG）。

glog默认将日志文件存储到/tmp目录下，文件的命名方式为<program name>.<hostname>.<user name>.log.<severity level>.<date>.<time>.<pid>，比如/tmp/hello_world.example.com.hamaji.log.INFO.20080709-222411.10474

## 配置标记
可以通过三种方式对glog进行配置，1、程序执行前，通过shell脚本设置环境变量进行配置。2、如果程序中集成了gflags，则可以通过gflags进行配置。3、通过glog的API进行配置。我们使用第三种方式进行配置。较常使用的几个配置项如下：

* **logtostderr(bool, default=false)**

	将日志消息全部输出到标准输出（打印到终端），代替输出到日志文件。

* **stderrthreshold(int, default=2, which is ERROR)**
	
	大于等于该等级的日志消息将同时输出到标准输出和日志文件中。

* **minloglevel(int, default=0, which is INFO)**
	
	大于等于该等级的日志消息会被输出。

* **log_dir(string, default="")**
	
	如果指定了该配置项，日志文件将会保存到该目录中。该条配置应该在glog初始化前被执行，否则无效，日志仍然保存在默认路径中。

* **v(int, default=0)**
	
	VLOG(m)，当m小于等于设定值时，该条日志消息才会被输出。详见下文自定义等级。

* **vmodule(string, default="")**
	
	模块的v等级，相关模块中的VLOG，只有小于等于该模块v等级的日志消息才会被输出。

通过对FLAGS_开头的全局变量进行赋值可以进行配置，比如：

```c++
LOG(INFO) << "file";   
// Most flags work immediately after updating values.   
FLAGS_logtostderr = 1;   
LOG(INFO) << "stderr";   
FLAGS_logtostderr = 0;   
// This won't change the log destination. If you want to set this   
// value, you should do this before google::InitGoogleLogging .   
FLAGS_log_dir = "/some/log/directory";   
LOG(INFO) << "the same file";
```

## 条件日志
控制日志消息当某种条件达成时才输出：

```c++
LOG_IF(INFO, num_cookies > 10) << "Got lots of cookies";
LOG_EVERY_N(INFO, 10) << "Got the " << google::COUNTER << "th cookie";
LOG_IF_EVERY_N(INFO, (size > 1024), 10) << "Got the " << google::COUNTER << "th big cookie";
LOG_FIRST_N(INFO, 20) << "Got the " << google::COUNTER << "th cookie";
```

## Debug模式支持
使用Debug模式的宏，如果在Release模式下，这些日志消息将不会被执行。

```c++
DLOG(INFO) << "Found cookies";
DLOG_IF(INFO, num_cookies > 10) << "Got lots of cookies";
DLOG_EVERY_N(INFO, 10) << "Got the " << google::COUNTER << "th cookie";
```

## CHECK宏
提供了一种检查到条件不匹配时中止程序的方法，与C的assert宏类似。但与assert不同的是，它不被NDEBUG所控制，因此无论何种编译模式下，它都会被执行。

```c++
CHECK(fp->Write(x) == 4) << "Write failed!";
CHECK_EQ(string("abc")[1], 'b');
```

## 自定义等级
glog提供VLOG宏，使用户可以自定义严重性等级，比如：

```c++
VLOG(1) << "I'm printed when you run the program with --v=1 or higher";
VLOG(2) << "I'm printed when you run the program with --v=2 or higher";
```
VLOG消息总是归属于INFO等级类型的日志消息。小于等于v值（负数也可以，但必须是整数）的日志消息才会被输出，比如--v=3，VLOG(3)、VLOG(1)都会输出，VLOG(4)不会输出。这种特性与标准等级刚好相反，当--minloglevel=1时，大于等于该等级的日志消息才会被输出，即WARNING、ERROR、FATAL会被输出。

```c++
//FLAGS_vmodule = "gps=2";  //实际中发现没有该全局变量 ，glog 0.3.5，使用另一接口代替 
google::SetVLOGLevel("gps", 2);
google::SetVLOGLevel("gfs*", 1);
FLAGS_v = 0;
``` 
在gps.{h,cc}中输出小于等于2的VLOG。在gfs开头的源文件中输出小于等于1的VLOG。其他情况打印小于等于0的VLOG

## 错误信号处理
当特定的信号导致程序崩溃时，glog库可以打印出有用的出错信息，通过google::InstallFailureSignalHandler()可以注册信号，出错信息示例：

```sh
*** Aborted at 1225095260 (unix time) try "date -d @1225095260" if you are using GNU date ****** SIGSEGV (@0x0) received by PID 17711 (TID 0x7f893090a6f0) from PID 0; stack trace: ***PC: @           0x412eb1 TestWaitingLogSink::send()    @     0x7f892fb417d0 (unknown)    @           0x412eb1 TestWaitingLogSink::send()    @     0x7f89304f7f06 google::LogMessage::SendToLog()    @     0x7f89304f35af google::LogMessage::Flush()    @     0x7f89304f3739 google::LogMessage::~LogMessage()    @           0x408cf4 TestLogSinkWaitTillSent()    @           0x4115de main    @     0x7f892f7ef1c4 (unknown)    @           0x4046f9 (unknown)
```

## 封装自定义LOG宏
glog实际应用中，我们需要在Debug模式下将所有日志同时输出到标准输出和日志文件，在Release模式下只输出大于等于WARNING的消息到标准输出和日志文件，并且在Release模式下FATAL不会终结程序，根据这些需求，我们封装了自己的LOG宏：

```c++
#define AEDEBUG    DLOG(INFO)
#define AEWARN     LOG(WARNING)
#define AEERROR    LOG(ERROR)
#define AEFATAL    LOG(DFATAL)
```

## 简单benchmark测试
在x86平台使用google benchmark库对glog的日志接口进行性能测试，结果如下：

```sh
--------------------------------------------------
Benchmark           Time           CPU          Iterations
--------------------------------------------------
BM_glog         39949 ns       3586 ns     100000
```
以上测试结果是对

```c++
LOG(INFO) << "this is a benchmark test for glog";
```
执行10万次的平均系统时间和平均CPU时间。该语句同时执行输出到标准输出和日志文件中。