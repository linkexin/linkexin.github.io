---
title: 通过 Component 获得一个 UIView
---

## 关于 ComponentKit
`ComponentKit` 是 `Facebook` 发布的一个受 `React` 启发而产生的 `iOS` 视图框架，最开始用于 `Facebook` 的 `News Feed`，现在已经开始作为整个 `Facebook` 的 `iOS` 开发框架使用。
更多：[官方文档](http://componentkit.org/)、[一个中文介绍文档](https://segmentfault.com/a/1190000002625560)

-------

`CK` 提供了一个可以根据 `component` 来构造布局的 `view`： `CKComponentHostingView`，这是一个 UIView 的子类。

## API

#### 初始化方法
```
/** Designated initializer. */
- (instancetype)initWithComponentProvider:(Class<CKComponentProvider>)componentProvider
                        sizeRangeProvider:(id<CKComponentSizeRangeProviding>)sizeRangeProvider;
```
* 第一个参数传入一个实现了 `CKComponentProvider` 协议的类，关于 `CKComponentProvider` 具体可见 [在列表中使用 CK](https://linkexin.github.io/notes/%E5%9C%A8%E5%88%97%E8%A1%A8%E4%B8%AD%E4%BD%BF%E7%94%A8-ComponentKit)

* 第二个参数传入一个计算大小的约束，比如说传入：

```
[CKComponentFlexibleSizeRangeProvider providerWithFlexibility:CKComponentSizeRangeFlexibleHeight]
```
表示这个 `view` 宽度固定，高度自由变化。

#### 赋值方法

```
/** Updates the model used to render the component. */
- (void)updateModel:(id<NSObject>)model mode:(CKUpdateMode)mode;

/** Updates the context used to render the component. */
- (void)updateContext:(id<NSObject>)context mode:(CKUpdateMode)mode;
```

`CKComponentProvider` 协议的方法是需要传入两个参数的：

```
+ (CKComponent *)componentForModel:(id<NSObject>)model context:(id<NSObject>)context
```

调用上面的两个方法后，相应的值就会作为参数传入到下面的方法中。调用这两个方法中的任意一个方法都会触发 `ckdatasource` 调用 `componentForModel` 方法来获得相应的 `component`。

#### 大小计算
需要注意的一点是，`hostingView` 是不会去设置自己的大小的，我们需要调用 `sizeThatFits：`方法来获得 `view` 的大小，然后再设置。

```
CKComponentHostingView *hostingView = [[CKComponentHostingView alloc]
                                       initWithComponentProvider:self.class
                                       sizeRangeProvider:[CKComponentFlexibleSizeRangeProvider providerWithFlexibility:CKComponentSizeRangeFlexibleHeight]];
        
CGSize size = [hostingView sizeThatFits:CGSizeMake(self.view.frame.size.width, CGFLOAT_MAX)];//layout Size
[self.view addSubview:hostingView];
// 因为 CKComponentHostingView 其实是个 view，所以可以用 autolayout 来布局
[hostingView mas_makeConstraints:^(MASConstraintMaker *make) {
   make.top.mas_equalTo(100);
   make.left.right.mas_equalTo(0);
   make.height.mas_equalTo(size.height);
}];
```

#### 代理方法
`CKComponentHostingView` 有一个代理 `CKComponentHostingViewDelegate`，代理方法只有一个：

```
- (void)componentHostingViewDidInvalidateSize:(CKComponentHostingView *)hostingView
```

当 `hostingView` 的大小发生变化的时候会调用此方法（当再次调用 `updateModel` 方法，会重新生成一个 `component`，这时因为传入 `model` 不同，所以通过 `component` 生成的 `view` 也会变化，其中就包括大小的变化）
上面提到过，`hostingView` 不会设置自己的高度，所以这里我们可以再计算一遍大小，再赋一遍值。


