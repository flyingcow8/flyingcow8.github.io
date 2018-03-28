---
title: "macOS上统计代码覆盖率"
date: 2015-07-15 15:11:00
categories: Programing
tags:
  - xcode
---

## 前言
Mac OS X下Xcode 6集成开发环境，可以统计代码的覆盖率情况，然后配合开源工具XcodeCoverage来生成HTML覆盖率报表。

Xcode使用GCOV对iOS和OS X的app或者库做覆盖率测试。当app编译完成后，Xcode会产生.gcno文件；当app运行退出后，会产生.gcda文件。XcodeCoverage调用LCOV工具，并使用.gcno和.gcda文件生成覆盖率统计报表。总体过程与Linux下覆盖率统计过程类似。
<!-- more -->
## 操作步骤
### 1 
首先，为了区别Debug和Release版本，我们新建一个叫做GCov_Build的编译配置文件，该文件从debug版本的文件duplicate得到。方法是在项目编辑区选择你的工程，点击上方的Info标签，在Configurations部分点击+按键，然后在弹出菜单中选择Duplicate "Debug" configuration。本文为该文件取名为GCov_Build。
![1]({{ site.url }}/assets/images/xcode1.png)

### 2 
在项目编辑区选中工程后，点击Build Settings标签页，按照下图所示进行配置
![2]({{ site.url }}/assets/images/xcode2.png)

### 3
编译工程并生成.gcno文件。点击Product>Scheme>Edit Scheme，左侧选择Run，右侧Info标签页中Build Configuration选择GCov_Build：
![3]({{ site.url }}/assets/images/xcode3.png)
然后选择Product>Build来编译整个工程，编译器会在编译路径下生成.gcno文件。点击Window>Organizer，Derived Data就是编译目录，点击旁边的箭头可以打开该目录：
![4]({{ site.url }}/assets/images/xcode4.png)
本例中，.gcno文件位于/Users/wangwangchuanyan/Library/Developer/Xcode/DerivedData/BeaconReceiver-alawxertnsbbpobsckmkwzzlswaz/Build/Intermediates/BeaconReceiver.build/Debug-iphoneos/BeaconReceiver.build/Objects-normal/armv7/ 目录中。

### 4 
运行应用程序，产生.gcda文件。对于模拟器调试的情况，则.gcda文件生成在.gcno文件同一目录中。对于真机调试的情况，由于存在沙盒机制，.gcda文件无法保存到.gcno目录下，因此我们需要人为地保存.gcda到沙盒内指定目录，然后将这些文件拷贝到mac上的.gcno文件的目录下。当应用程序正常退出时，xcode才会将覆盖率数据写到.gcda文件中。对于OS X程序，只要通过Quit菜单退出即可。iOS应用默认不会退出，因此需要临时修改应用使得覆盖率数据强制写到.gcda文件中，有两种方式可以实现该目的。第一种方式，在工程的Info.plist文件中设置Application does not run in background键为YES，这样的话一旦按下Home键，应用便会退到后台。第二种方式是在AppDelegate添加代码的方式。推荐使用第二种方式。

对于模拟器的情况，添加代码：

```oc
#import "AppDelegate.h"
#include <stdio.h>
 
extern void __gcov_flush();
@implementation AppDelegate
 
- (void)applicationDidEnterBackground:(UIApplication *)application
{
  __gcov_flush();
}
 
......
@end
```
app退出后，.gcda文件生成在.gcno文件同一目录中，比如/Users/wangwangchuanyan/Library/Developer/Xcode/DerivedData/BeaconReceiver-alawxertnsbbpobsckmkwzzlswaz/Build/Intermediates/BeaconReceiver.build/Debug-iphoneos/BeaconReceiver.build/Objects-normal/armv7/

对于真机的情况，添加代码：

```oc
#import "AppDelegate.h"
#include <stdio.h>
 
extern void __gcov_flush();
@implementation AppDelegate
 
- (void)applicationDidEnterBackground:(UIApplication *)application
{
  NSArray *paths = NSSearchPathForDirectoriesInDomains(NSDocumentDirectory,   NSUserDomainMask, YES); 
  NSString *documentsDirectory = [paths objectAtIndex:0]; 
  setenv("GCOV_PREFIX", [documentsDirectory cStringUsingEncoding:NSUTF8StringEncoding], 1); 
  setenv("GCOV_PREFIX_STRIP", "13", 1);
     __gcov_flush();
}

......
@end
```
第一个setenv的意思是将数据的根目录设置为app的Documents; 第二个setenv的意思是strip掉一些目录层次，因为覆盖率数据默认会写入一个很深的目录层次。完成测试后，生成的.gcda文件会保存在iOS设备上app的Documents目录下，可以通过MAC自带的iTunes软件或者第三方工具将这些文件复制到MAC上.gcno的目录中。

### 5
将覆盖率数据转换为可视化的HTML统计报表文档。本文使用开源工具XcodeCoverage，该工具可以得到一组HTML文件，方便地查看每个目录下的每个类和函数的执行情况。XcodeCoverage是LCOV和一些脚本的集合。首先下载该工具，并解压缩为XcodeCoverage文件夹，将该文件夹放到项目.xcodeproj文件所在的文件夹内
![5]({{ site.url }}/assets/images/xcode5.png)
然后在工程的主target中添加一个Run Script build phase，执行XcodeCoverage/exportenv.sh。可以将该脚本直接拖到红色方框的区域，或者直接输入XcodeCoverage/exportenv.sh目录即可。
![6]({{ site.url }}/assets/images/xcode6.png)
使用XcodeCoverage还需要在工程中添加一个xcconfig文件，如果工程中已有一个xcconfig文件，只需要在该文件中添加#include "XcodeCoverage/XcodeCoverage.xcconfig"。如果没有，则把XcodeCoverage.xcconfig拖进工程即可。然后在PROJECT的info标签页，在Configurations下的GCov_Build，配置为XcodeCoverage。
![7]({{ site.url }}/assets/images/xcode7.png)
直接双击XcodeCoverage目录中getcov，或者在Terminal中命令行运行该脚本，即可生成HTML覆盖率报告。

getcov是一个shell脚本，执行的时候可以传递参数进行控制，具体可参考XcodeCoverage的README文档。同时，可以进行一些必要的修改。在exclude_data函数中，可以排除一些不想要统计覆盖率的代码，比如本例中不想统计第三方库的目录，该库位于library子目录中，可以这样来排除：

```sh
LCOV --remove ${LCOV_INFO} "library/*" -d "${OBJ_DIR}" -o ${LCOV_INFO}
```
在generate_html_report函数中，我们可以更改生成报告的路径，默认的路径位于编译目录下的一个很深的目录中，很难找到，我们把它改到XcodeCoverage目录下的report文件夹中：
"${LCOV_PATH}/genhtml" --output-directory "${scripts}/report" "${LCOV_INFO}"

另外每次覆盖率测试最好用一个全新的build，因此需要将上次build的中间文件全部删除，可以打开Xcode的Product菜单，按住Option键，点击Clear Build Folder…。然后运行XcodeCoverage目录中的cleancov，重新编译并运行app，测试退出后记得将新产生的.gcda文件复制到.gcno目录中。运行getcov来生成HTML报告，文件名为index.html。
![8]({{ site.url }}/assets/images/lcov.png)

## 参考
1. [Configuring Xcode for Code Coverage](
https://developer.apple.com/library/ios/qa/qa1514/_index.html)
2. [xcode进行代码覆盖率测试](
http://www.cnblogs.com/longhuihu/p/4032611.html?utm_source=tuicool)
