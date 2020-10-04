---
title: Xcode Memory Debugger 
---
 
## 前言
iOS 内存分析除了使用 Instrument 以外，Xcode 中也有 Memory Debugger，其实很早以前尝试使用过 Xcode Memory Debugger，但是当时一头雾水。现在稍微有些了解了，记录下来。
Memory Debugger 可以在 XCode 的可视化界面中分析使用，也可以将 .MEMGRAPH 文件导出，再借由命令行分析。

## 图形界面分析
Xcode Memory Debugger 的入口为 Xcode 底下工具条的 Debug Memory Graph 按钮，运行项目后查看内存图形，可以看到如下界面：
![-w250](https://res.cloudinary.com/dp1pheuq7/image/upload/v1601794962/Xcode_Memory_Debugger_1_gbxuhc.png)
右边有一些信息展示，但是可以看到 Backtrace 一栏无法显示有效信息
![-w250](https://res.cloudinary.com/dp1pheuq7/image/upload/v1601794947/Xcode_Memory_Debugger_2_j3ykgk.png)
为了回溯内存分配的堆栈，需要再设置中将 Malloc Stack 选项勾上：
![](https://res.cloudinary.com/dp1pheuq7/image/upload/v1601795033/Xcode_Memory_Debugger_3_gbstj1.png)
再运行就可以看到右侧的堆栈信息：
![](https://res.cloudinary.com/dp1pheuq7/image/upload/v1601794949/Xcode_Memory_Debugger_4_on9oth.png)

界面左边有底部有两个按钮，选中左边的按钮，会筛选出所有有引用循环的地方
选中右边的按钮，会将系统内容过滤掉，只显示和项目有关的内容
![](https://res.cloudinary.com/dp1pheuq7/image/upload/v1601794969/Xcode_Memory_Debugger_5_v0pa0z.png)


## 命令行分析 .MEMGRAPH 文件
在 Memory Debugger 界面产生时 Xcode 也同时生成了一个 .MEMGRAPH 文件，可以将其导出：![](https://res.cloudinary.com/dp1pheuq7/image/upload/v1601794966/Xcode_Memory_Debugger_6_betlvr.png)

接下来就可以使用命令行来分析 .MEMGRAPH 文件：
### vmmap
vmmap 通过将进程所占虚拟内存信息打印出来，以提供一些关于进程内存消耗更高级别的分析

> vmmap App.memgraph  
能查看所有内存区域的具体信息
首先是 non-writeable region，比如可执行文件、程序文本等：
![](https://res.cloudinary.com/dp1pheuq7/image/upload/v1601794969/Xcode_Memory_Debugger_7_sbrgm8.png)  
接着是 writeable region，这里就是 app 进程的堆所在的位置：
![](https://res.cloudinary.com/dp1pheuq7/image/upload/v1601794959/Xcode_Memory_Debugger_7.1_ptmtlw.png)  

> vmmap —summary App.memgraph   
可以看到一些 summary info 和不同类型的虚拟内存所占不同内存区块的大小，如果我们关注内存问题，应该关注 DIRTY SIZE 和 SWAPPED SIZE 这两列，分别表示脏内存大小和交换内存大小
![](https://res.cloudinary.com/dp1pheuq7/image/upload/v1601795032/Xcode_Memory_Debugger_8_oesxdb.png)

### leaks
Leaks 工具会在运行时追踪到堆中没有根的对象，如果看到一个对象泄漏了，那么一定存在脏内存无法被释放
假如我们在 XCode 可视化界面看到了这样一个内存泄漏：
![](https://res.cloudinary.com/dp1pheuq7/image/upload/v1601795023/Xcode_Memory_Debugger_9_smatay.png)

用 leaks 命令，在命令行中也能看到对应的信息，而且不仅是泄漏的对象，还有造成泄漏的回溯堆栈(前提是启用了 Malloc Stack)，太贴心了
![](https://res.cloudinary.com/dp1pheuq7/image/upload/v1601794978/Xcode_Memory_Debugger_10_r2fdrn.png) 

### heap
有时候我们使用 vvmap 后发现堆内存很大，但是仅此而已，不会有跟多的信息
这时候可以借助 heap 命令，heap 可以帮助显示堆中的内存分配，用于追踪非常复杂的分配

> heap App.memgraph  
展示类名、这类对象的数量、平均大小、总大小
看这张图可以发现，heap 默认是按数量来排序的，但是按此排序数量最多的大多是是系统对象，这样我们其实看不出问题在哪里
![](https://res.cloudinary.com/dp1pheuq7/image/upload/v1601795024/Xcode_Memory_Debugger_11_shx9oe.png) 

我们可以传递参数，让 heap 按照 size 来排序：
> heap —sortBySize App.memgraph   
![](https://res.cloudinary.com/dp1pheuq7/image/upload/v1601795026/Xcode_Memory_Debugger_12_ehpki5.png) 

上图可以看到 NSConcreteData 占了非常多内存，但这还不够，还需要知道这些对象是怎么形成的

> heap App.memgraph -addresses all | <classes-pattern>   
当通过 -addresses 将具体的类名传递给 heap 指令时，heap 会罗列出所有指定类的每个实例的地址，有了这些地址，我们就能进一步研究这些对象是哪里来的了
![](https://res.cloudinary.com/dp1pheuq7/image/upload/v1601795026/Xcode_Memory_Debugger_13_lil4my.png) 
 
> malloc_history <memgraph> <address>   
显示具体 .memgraph 文件的具体对象地址的堆栈
我们将上面 NSConcreteData 的其中一个地址输入，得到回溯记录，并且在堆栈中找到了和 App 相关的方法
![](https://res.cloudinary.com/dp1pheuq7/image/upload/v1601795038/Xcode_Memory_Debugger_14_suanyq.png) ![](https://res.cloudinary.com/dp1pheuq7/image/upload/v1601795031/Xcode_Memory_Debugger_15_twschh.png)
综上，我们就通过命令行追踪到了大内存消耗的具体原因

## 总结
上面提到的几种指令，各有不同的作用：
malloc_history：关注对象的创建
Leaks：关注对象引用，内存泄漏
vmmap、heap：关注一个属于或者一个实例有多大

## 参考
[iOS Memory Deep Dive - WWDC 2018 - Videos - Apple Developer](https://developer.apple.com/videos/play/wwdc2018/416/)
