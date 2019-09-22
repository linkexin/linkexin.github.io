--- 
title: URLProtocol 注册方式的细节问题
--- 

## 前言
苹果提供了 URLProtocol 让开发者可以介入 URL Loading System 做一些定制操作。

## 注册 protocol
注册 protocol 有两种方式
1. [NSURLProtocol registerClass:protocol.class]
2. 手动修改 protocolClasses
``` 
NSURLSessionConfiguration *c = [NSURLSessionConfiguration defaultSessionConfiguration];
NSMutableArray *a = c.protocolClasses.mutableCopy;
[a insertObject:[protocol1 class] atIndex:0];
c.protocolClasses = a;
```
在网上查资料，普遍说的是第一种方式，我本以为两种方式只需要选择一种就可以，却在实际开发中发现针对不同类型的 configuration 会有不一样的表现。

#### 针对 sharedSession
[NSURLSession sharedSession] 的 configuration 类型是 defaultSessionConfiguration，通过 [NSURLProtocol registerClass:protocol.class] 注册的 protocol 会作用在 [NSURLSession sharedSession] 的 configuration 上，发起请求时会按照 register 的倒叙顺序来逐个确认 protocol 是否 handle 当前的请求
![](URLSessionConfiguration%20%E6%B3%A8%E5%86%8C%E6%96%B9%E5%BC%8F%E7%9A%84%E7%BB%86%E8%8A%82%E9%97%AE%E9%A2%98/%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A7%202019-09-22%20%E4%B8%8B%E5%8D%884.20.32.png)

#### 针对 defaultSessionConfiguration/ephemeralSessionConfiguration
每次调用 [NSURLSessionConfiguration defaultSessionConfiguration] 都会返回一个新的 config 对象，只是这个对象的配置是 default 的，每次调用 copy 方法也会生成一个新的 config 对象
defaultSessionConfiguration 不受  [NSURLProtocol registerClass:protocol1.class] 方法的影响：
![](URLSessionConfiguration%20%E6%B3%A8%E5%86%8C%E6%96%B9%E5%BC%8F%E7%9A%84%E7%BB%86%E8%8A%82%E9%97%AE%E9%A2%98/%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A7%202019-09-22%20%E4%B8%8B%E5%8D%884.29.11.png)

我们需要使用方式 2 来注册：
![](URLSessionConfiguration%20%E6%B3%A8%E5%86%8C%E6%96%B9%E5%BC%8F%E7%9A%84%E7%BB%86%E8%8A%82%E9%97%AE%E9%A2%98/%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A7%202019-09-22%20%E4%B8%8B%E5%8D%884.32.50.png)

## protocol 的调用顺序
无论是对于那种注册方式，protocol 的调用顺序都是按照 protocolClasses 数组中的顺序，从前往后依次调用各个 protocol 的 canInitWith 方法，找到第一个可以 handle 的 protocol，后续的 protocol 就不会再调用了。
![](URLSessionConfiguration%20%E6%B3%A8%E5%86%8C%E6%96%B9%E5%BC%8F%E7%9A%84%E7%BB%86%E8%8A%82%E9%97%AE%E9%A2%98/%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A7%202019-09-22%20%E4%B8%8B%E5%8D%885.13.43.png)

## 关于 background sessions
官方文档 [protocolClasses - NSURLSessionConfiguration | Apple Developer Documentation](https://developer.apple.com/documentation/foundation/nsurlsessionconfiguration/1411050-protocolclasses?language=objc) 中的 note 部分有提到：
```
You cannot use custom NSURLProtocol subclasses in conjunction with background sessions.
```
查了下资料：Background sessions are similar to default sessions, except that a separate process handles all data transfers.


## 总结
针对不同的类型的 config，注册 protocol 的方式不同，但是 protocol 的调用逻辑是一致的

