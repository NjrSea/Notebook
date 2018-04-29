## 背景
随着抖音项目的发展，iOS工程规模、开发人数也随之急剧增长。随着去年公司对Musical.ly的收购，Musical.ly的code base也随之迁移到了抖音，目前代码库承载的是抖音、Tik Tok和Musical.ly三款应用和北京、上海两地30多人的团队开发工作。如何提高工程效率和工程质量成为我们团队的一个很重要的问题，比如提交代码导致编译不通过的问题，这可能会同时影响开发效率、CI打包和测试进度；或者是有问题的代码没有在code review、测试、灰度中发现。针对这些问题，我们引入了GitLab CI机制。开发人员提交代码到GitLab，首先会通过一些列的处理，称之为[Pipeline](https://docs.gitlab.com/ee/ci/pipelines.html)，如果提交的代码未通过Pipeline，那么代码就不允许自动合入。比如提交的代码在Tik Tok上有编译问题而在其他两个APP编译通过，则构建步骤失败，这次提交代码的Pipeline执行结束，并通过lark发布CI Pipeline执行结果信息，通过各种手段提前发现问题，提高了团队整体的工作效率。除了通过构建发现编译错误之外，在CI的nightly build的中还有一个十分重要的环节，那就是静态分析。

那么什么是静态分析呢？

静态分析是指在不运行计算机程序的条件下，进行程序分析的方法。有些程序分析需要在程序运行时才能进行，这种程序分析称为动态程序分析。大部分的静态程序分析的对象是针对特定版本的源代码，也有些静态程序分析的对象是目标代码。静态程序分析一词多半是指配合静态程序分析工具进行的分析，人工进行的分析一般称为程序理解或代码审查。

静态程序分析的复杂程度依所使用的工具而异，简单的只考虑个别语句及声明的行为，复杂的可以分析程序的完整源代码。不同静态程序分析技术对分析得到的信息的用途也有所不同，简单的可以是高亮标识可能存在的代码错误（如lint），复杂的可以是形式化方法，也就是用数学的方式证明程序的某些行为匹配其设计规约。

目前在iOS工程中常用的静态分析工具有Clang Static Analyzer，OCLint和Infer：

```
***Clang Static Analyzer***

Clang Static Analyzer既XCode中集成的Analyze分析工具，可以通过Xcode—>Product—>Analyze找到。


***OCLint***

OCLint是一款开源的静态分析工具，用于C，C++和Objective-C代码的静态分析。

***Infer***

Facebook 的 Infer 是一个静态分析工具。Infer 可以分析 Objective-C， Java 或者 C 代码，报告潜在的问题，同时防止应用崩溃和性能低下。 
```

经过调研，我总结了OCLint和Infer各自的优缺点：

***OCLint***

* 优点
	* 接入简单
	* 定制规则成本低
	* 分析结果以html文件形式输出，可读性强

* 缺点
	* 捕捉bug的类型有限
	* 开源社区不活跃

***Infer***

* 优点
	* 捕捉bug类型多
	* 效率高，规模大，几分钟能扫描数千行代码
	* 分解分析，整合输出结果（能将代码分解，小范围分析后再将结果整合在一起，兼顾分析的深度和速度）
	* 开源社区氛围好，项目维护人员响应问题速度快

* 缺点
	* 接入难度高
	* 规则定制工作量大
	* 分析结果没有友好的格式输出，需要自行开发


出于对分析速度和质量的考虑，最终选择在项目中使用Infer作为静态分析工具。


## Infer工作原理

不管是分析那种语言，Infer运行时，分为两个主要阶段：
### 1.捕获阶段
---

Infer捕获编译命令，将文件翻译成Infer内部的中间语言。

这种翻译和编译类似，Infer从编译过程获取信息，并运行翻译。这就是我们调用Infer时带上编译命令的原因了，比如：```infer -- clang -c file.c```, ```infer -- xocdebuild File.m ```。结果就是文件照常编译，同时被Infer翻译成中间语言，留作第二阶段处理。特别注意的就是，如果没有文件被编译，那么也没有任何文件被分析。

Infer把中间文件存储在结果文件夹中，一般来说，这个文件夹会在Infer的目录下创建，命名是infer-out/。当然，你也可以通过-o选项来自定义文件夹名字：

```
infer -o /tmp/out -- xcodebuild test.m
```

### 2.分析阶段

---

在分析阶段，Infer分析infer-out/下的所有文件。分析时，会单独分析每个方法和函数。

在分析一个函数的时候，如果发现错误，将会停止分析，但这不影响其他函数的继续分析。

所以你在查问题的时候，修复输出的错误值后，需要继续运行Infer进行检查，直到确认所有问题都已修复。

错误除了会显示在标准输出之外，还会输出到文件infer-out/bug.txt中，我们考虑这些问题，仅显示最有可能存在的。

在结果文件夹中，同时还有一个csv文件report.csv，这里包含了所有Infer产生的信息，包括：错误、警告和信息。


## 错误类型

Infer可以检测潜在的错误类型有很多种，下面我看一下在iOS开发中常见的错误类型。

#### Resource leak(资源泄漏)

resources通常指文件、socket、连接等用后需要关闭的资源。这个错误类型在使用过的资源没有被释放时会出现。

示例：

```
- (void)resource_leak_bug {
    FILE *fp;
    fp=fopen("c:\\test.txt", "r"); // file opened and not closed.
}
```

#### Memory leak(内存泄漏)

开发者自己申请的内存没有被合理释放时，会报这个错误

示例：

```
- (void)memory_leak_bug {
    struct Person *p = malloc(sizeof(struct Person));
}
```

```
- (void)memory_leak_bug_cf {
    CGPathRef shadowPath = CGPathCreateWithRect(self.inputView.bounds, NULL); //object created and not released.
}
```

#### Retain cycle(循环引用)

iOS开发众所周知的bug，在此不再赘述。

#### Null Dereference（空指针解引用）

Infer会找到隐藏在C、Objective-C和Java代码中的空指针解引用。在以上三种语言中对空指针进行解引用会导致crash。

示例:

c语言

```
struct Person {
  int age;
  int height;
  int weight;
};
int get_age(struct Person *who) {
  return who->age;
}
int null_pointer_interproc() {
  struct Person *joe = 0;
  return get_age(joe);
}
```

objective-c

```
- (void)foo:(void (^)())callback {
    callback();
}

- (void)bar {
    [self foo:nil]; //crash
}
```

#### Parameter not null checked （参数未做非空检查）

这个问题仅在objective-c代码中会检查，它和Null dereference问题相似，infer找到了有Null Dereference（空指针解引用）错误的代码，但是没有找到可能出现问题的完整路径。

示例：

```
 - (int)foo:(A* a) {
      B b* = [a foo]; // sending a message with receiver nil returns nil
      return b->x; // dereferencing b, potential NPE if you pass nil as the argument a.
  }
```

或者参数是一个block：

```
 - (void)foo:(void (^)(BOOL))block {
      block(YES); // calling a nil block will cause a crash.
 }
```

一般的解决方案是添加一个非空的判断，或者开发者能确定这个方法不会传入空参数。当参数确认不会为空时，可以使用```nonnull ```修饰参数类型，这样infer就不会认为这是一段有问题的代码。

#### Bad pointer comparison（错误指针比较）

Infer只在objective-c代码中检查这类错误，把这类问题视为警告。在iOS中，有些类会对原有的值进行包装，比如```NSNumber```, 当代码中的```if```语句是用一个包装后的类型进行布尔运算做条件时，Infer就会报告这样的错误。试想以下代码：

```
void foo(NSNumber * n) {
  if (n) ...
}
```

这段代码中，```if```会以n这个```NSNumber *```类型的变量做为条件，但是开发者可能需要判断的是通过```[n integerValue]```方法得到变量```n```中包裹的整数值是否大于0。


Infer可以检查的错误类型还有很多，这里就不一一列举了。

## Infer接入

### 安装
安装xcpretty, xcpretty是一款格式化输出xcodebuild编译信息的工具，Infer会用到

```
gem install xcpretty
```

安装Infer

```
brew reinstall infer
```

### 命令介绍

首先，简单介绍一下infer的命令：

* ```infer capture [options]``` 捕获命令

* ```infer analyze [options]``` 分析命令

* ```infer run [options]``` 捕获后分析最后将分析结果输出的

* ```infer [options]``` 当infer后跟-- compile command时，表现和命令```run```相同，否则与```analyze```相同，其中compile command是具体执行编译的命令，如：```infer -- xcodebuild```

抖音工程中用到的infer命令如下：

```
infer --keep-going
--no-xcpretty 
--debug
--linters-ignore-clang-failures 
--skip-clang-analysis-in-path Pods 
-- xcodebuild 
-workspace Aweme.xcworkspace 
-configuration AwemeDebug 
-scheme Aweme 
-sdk iphonesimulator 
clean build
```

### 环境变量

INFER_ARGS：多余的参数可以通过INFER_ARGS变量提供给infer，格式需为以'^'符号分字符串，如：```INFER_ARGS=--debug^--print-logs infer```，赋值给环境变量后，与命令```infer --debug --print-logs```相同。

INFERCONFIG： 指定.inferconfig文件所在位置

```
.inferconfig

.inferconfig文件是infer命令的配置文件，它可以用于保存infer命令的选项。文件采用JSON格式，key是长格式的参数名去掉“--”，对应的value可以是以下几种:

1. 开关选项对应boolean类型
2. 整数选项对应integer类型
3. 字符串选项对应string类型
4. 路径选项对应string类型，并且是相对于.inferconfig的相对路径
```

INFER_STRICT_MODE: 严格模式，当为1时，当infer产生警告时便会退出。

### 接入中遇到的问题


在加```--no-xcpretty```参数之前，使用infer遇到了如下问题：

```
...
Build Succeeded
Starting translating 463 files 

*** ERROR: Failed to execute compilation command. Output:
clang: error: cannot specify -o when generating multiple output files
*** Infer needs a working compilation command to run.
..MANY OF THESE ERRORS...then...

...
*** ERROR: Failed to execute compilation command. Output:
clang: error: cannot specify -o when generating multiple output files
*** Infer needs a working compilation command to run.
..

Nothing to compile. Try running `xcodebuild -workspace myProject.xcworkspace -scheme myProject -configuration Debug -sdk iphonesimulator clean` first.

There was nothing to analyze.
```

在官方的[issue](https://github.com/facebook/infer/issues/759)中找到了项目维护者的回复：

>  Thanks all for testing. By default infer pipes xcodebuild output to xcpretty to get a compilation database. --no-xcpretty makes infer use a different integration that works by having xcodebuild call infer directly instead of calling clang. The latter used to be more fragile, but now it looks like it's the other way around.
**The xcpretty-based integration should be fixed, but in the meantime please use the --no-xcpretty workaround.**

加上```--no-xcpretty```后我们重新执行infer命令，控制台输出了如下错误：


```
In file included from /Users/paul/Desktop/Aweme/Aweme/Pods/HTSAccountService/HTSAccountService/Classes/Platform/QQ/NHAccountPlatformQQ.m:11:
In file included from /Users/paul/Desktop/Aweme/Aweme/Pods/Headers/Public/HTSSDKManager/TencentOpenAPI/TencentApiInterface.h:12:
/Users/paul/Desktop/Aweme/Aweme/Pods/Headers/Public/HTSSDKManager/TencentOpenAPI/TencentMessageObject.h:65:5: error: redefinition of enumerator 'ReqFromThirdAppShowContent'
ReqFromThirdAppShowContent,
^
In file included from /Users/paul/Desktop/Aweme/Aweme/Pods/HTSAccountService/HTSAccountService/Classes/Platform/QQ/NHAccountPlatformQQ.m:10:
In file included from /Users/paul/Desktop/Aweme/Aweme/Pods/Headers/Public/HTSSDKManager/TencentOpenAPI/TencentOAuth.h:12:
/Users/paul/Desktop/Aweme/Aweme/Pods/Headers/Public/HTSSDKManager/TencentOpenAPI/TencentApiInterface.h:12:9: note: '/Users/paul/Desktop/Aweme/Aweme/Pods/Headers/Public/HTSSDKManager/TencentOpenAPI/TencentMessageObject.h' included multiple times, additional include
site here
#import "TencentMessageObject.h"
^
In file included from /Users/paul/Desktop/Aweme/Aweme/Pods/HTSAccountService/HTSAccountService/Classes/Platform/QQ/NHAccountPlatformQQ.m:11:
/Users/paul/Desktop/Aweme/Aweme/Pods/Headers/Public/HTSSDKManager/TencentOpenAPI/TencentApiInterface.h:12:9: note: '/Users/paul/Desktop/Aweme/Aweme/Pods/Headers/Public/HTSSDKManager/TencentOpenAPI/TencentMessageObject.h' included multiple times, additional include
site here
#import "TencentMessageObject.h"
^

```

分析报错信息后，删除了.m文件中的多余头文件引入，再次尝试，编译通过，并进行静态分析，分析结果输出到了infer-out文件夹。

跑通整个分析流程后，将Infer加入CI流程，至此，抖音项目的接入完成。