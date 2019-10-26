--- 
title: Simple Ping in Swift - 预备芝士
---

## Swift 指针和内存操作
### 几种常用 Swift 指针类型
* unsafePointer
`unsafePointer<T>` 等同于 `const T *`
* unsafeMutablePointer
`unsafeMutablePointer<T>` 等同于 `T *`
* unsafeRawPointer
`unsafeRawPointer` 等同于 `const void *`，指向不应该修改的内存，无法修改所指向的内容
（tips：const void * a 指的是 (* a) 取出来的数是常量，而a本身是变量；Void * const a指的是(*a)取出来的数是变量，而a本身是常量）
* unsafeMutableRawPointer
`unsafeMutableRawPointer` 等同于 `void *`

### Swift Struct 的内存模型
涉及到 Size、Stride、Alignment 相关详见上一篇：[Size, Stride, Alignment · YUI 的严肃文](https://linkexin.github.io/notes/Size,-Stride,-Alignment)
通过操作内存来修改 Struct 类型实例属性值的几个具体方法：
* withUnsafeMutableBytes<ResultType>：用特定的指针类型来访问 data，指针可以访问的数据类型是 pointee 类型，UnsafeMutablePointer<UInt8> 就表示我们生成的指针访问的数据类型是 UInt8 类型的
* advanced：调用者必须遵循 BinaryInteger 协议 ，BinaryInteger 协议中有一个 bitwidth 方法，很字面的就是位宽，UInt8.bitWidth = 8，UInt16.bitWidth = 16，所以同样是 advanced(by: 1)，UInt8 往后移动了一个字节，UInt16 往后移动了两个字节
* pointee：从英语的层面理解，ee 结尾表示动作的承受者，也就是被指针指向内存地址的值

参考：[Swift 对象内存模型探究（一） - iOS - 掘金](https://juejin.im/entry/59156846a22b9d0058007283)

### inout 关键字
Swift 中的 inout 有点类似于 C 中的指针引用，但实际上Swift中inout只不过是::按值传递::，然后再写回原变量，而不是按引用传递：
```
An in-out parameter has a value that is passed in to the function, is modified by the function, and is passed back out of the function to replace the original value.
```

简单的🌰：
```
func inc(inout i: Int) {++i}

var x = 0
inc(&x)
print(x)	// 输出结果：“1”
```

按值传递的好处在于它远比使用引用安全，我们知道闭包都是通过引用的方式来持有变量的
举个 🌰：
```
func inc(inout i: Int) -> () -> Int {
	return { ++i }  // 闭包持有 inout 参数 i
}

var x = 0
let f = inc(&x)
print(f()) // 输出结果：“1”
print(x) // 输出结果：“0”
```
这段方法执行的过程：
1. x 的初始值为 0，x 作为 into 参数传入闭包，但是闭包并不是持有 x 本身，而是持有的 x 的一个副本假设为 x1
2. 返回后执行闭包，i 原本为 0，加完后变成 1，方法执行完后，x1 的值也会变成 1
3. x 没有被改动，所以还是 0

### UnsafeMutablePointer 是按引用传递
如果在函数声明中，参数是一个 UnsafeMutablePointer 的指针，那么传递参数的时候也要加上&，这和 inout 参数看上去用法类似，但实际上这里是按引用传递而不是按值传递。

## Swift 的高级运算符
不同于 C 语言中的数值计算，Swift 的数值计算默认是不可溢出的，溢出行为会被捕获并报错。但是 Swift 为我们提供了支持溢出的运算符，这样在溢出时就可以对有效位进行截断，从而表现的和 C 语言一致。
所有的溢出运算符都是以 & 开头的，Swift 为我们提供了 5 个 溢出运算符：
```
溢出加法 &+
溢出减法 &-
溢出乘法 &*
溢出除法 &/
溢出求余 &%
```
使用非溢出运算符：
```
var potentialOverflow = Int16.max
// potentialOverflow 等于 32767, 这是 Int16 能承载的最大整数
potentialOverflow += 1
// 出错
```
使用溢出运算符：
```
var willOverflow = UInt8.max
// willOverflow 等于UInt8的最大整数 255
willOverflow = willOverflow &+ 1
// 此时 willOverflow 等于 0
```

参考：[高级操作符 | 《The Swift Programming Language》中文版](https://numbbbbb.gitbooks.io/-the-swift-programming-language-/content/chapter2/24_Advanced_Operators.html)

## 用到的 Socket 库 API
* sockaddr_storage
Socket 编程中标准的地址结构体，128 个字节，足以存储IPv6地址，是 sockaddr 的拓展，因为在 sockaddr 诞生时期，还只有 IPv4，

* sendto
把数据报发给指定地址，在无连接的数据报 socket 方式下，由于本地 socket 并没有与远端机器建立连接，所以在发送数据时应指明目的地址，成功则返回实际传送出去的字符数，失败返回-1，错误原因存于errno 中。
sendto() 函数原型为：  
```
int sendto (int s, const void *buf, int len, unsigned int flags, const struct sockaddr *to, int tolen);

\s：           socket描述符。
\buf：         数据报缓存地址。
\len：         数据报长度。
\flags：       该参数一般为0。
\to：          struct sockaddr_in 类型，指明数据发往哪里报。
\tolen：       对方地址长度，一般为：sizeof(struct sockaddr_in)。
```
* recvfrom
接收远程主机经指定的 socket 传来的数据，成功则返回接收到的字符数，失败则返回-1，错误原因存于errno中。
```
int recvfrom(int s, void *buf, int len, unsigned int flags, struct sockaddr *from, int *fromlen);

\s：           socket描述符。
\buf：         接收数据缓冲区。
\len：         缓冲区长度。
\flags：       该参数一般为0。
\from：        指向装有源地址的缓冲区。
\fromlen：     struct sockaddr_in类型，指向from缓冲区长度值。
```

其中  sockaddr_in 结构体如下：
```
/*
 * Socket address, internet style.
 */
struct sockaddr_in {
	__uint8_t       sin_len;
	sa_family_t     sin_family; // 地址族
	in_port_t       sin_port; // 端口
	struct  in_addr sin_addr; // IP 地址
	char            sin_zero[8];
};
```

## ping 的完整过程
PING 是一个应用层服务，用来检测两个主机之间的连通性，Ping 使用了 ICMP 回送请求与回送回答报文，ping 是应用层直接使用 IP 层 ICMP 的一个例子，它没有通过运输层的 TCP 或者 UDP
[-w250](https://res.cloudinary.com/dp1pheuq7/image/upload/v1572084520/Ping_%E5%AE%8C%E6%95%B4%E8%BF%87%E7%A8%8B_cwr996.png)