---
title: 在 Component 中使用 UIView
---

## 关于 ComponentKit
`ComponentKit` 是 `Facebook` 发布的一个受 `React` 启发而产生的 `iOS` 视图框架，最开始用于 `Facebook` 的 `News Feed`，现在已经开始作为整个 `Facebook` 的 `iOS` 开发框架使用。
更多：[官方文档](http://componentkit.org/)、[一个中文介绍文档](https://segmentfault.com/a/1190000002625560)

-------

我们现在有一个现成的自定义的 UIView，怎么将其放入 component 结构中呢？根据个人的使用情况，列举两个方法：

* [CKComponent newWithView:{} size:{}]
* CKStatefulViewComponent & CKStatefulViewComponentController


## newWithView 方法
这是所有 component 的最基础的父类 CKComponent 提供的一个方法，其方法定义如下：

```
+ (instancetype)newWithView:(const CKComponentViewConfiguration &)view
                       size:(const CKComponentSize &)size;
```

看这个接口可以看出来，并不是将一个 UIView 对象直接传入，而是需要通过 CKComponentViewConfiguration 来描述一下这个 view，然后 CK 会根据描述来创建和设置相应的 view。

那么怎么描述呢？CKComponentViewConfiguration 的定义如下：

```
struct CKComponentViewConfiguration {
    CKComponentViewClass viewClass;
    std::unordered_map<CKViewComponentAttribute, id> attributes;
};
```

这里暂时不用去关心 CKComponentViewClass 和 CKViewComponentAttribute 是什么，大多数情况下，CKComponentViewClass 只需要传入一个 UIView 的 class 就可以，比如 [UIButton class]，[UIImageView class] 等；而 CKViewComponentAttribute 在多数的情况下传入一个 SEL

结合上面说的这点，可以直接看例子1：

```
[CKComponent
newWithView:{
    [UIImageView class],
    {
        {@selector(setImage:), image},
        {@selector(setContentMode:), @(UIViewContentModeCenter)}
    }
}
size:{image.size.width, image.size.height}];
```

上面这段代码我们向 CK 描述了一个 view，这个 view 是一个 UIImageView，并且我们设置了它的 image 和 contentMode。
这段代码中 CK 会帮我们做三件事：
1. 在 component 创建的时候自动创建或者重用 UIImageView
2. 自动调用 setImage 和 setContentMode，参数就是我们后面给定的参数
3. 在重用 UIImageView 的时候，如果所给定的参数和上次设置的时候是一样的，那 CK 是不会再去调用响应的设置方法的

✨有两个注意点：
1. CKComponentViewConfiguration 中 attributes 的 value 是 id 类型的，注意转换
2. 调用 [CKComponent newWithView:{} size:{}] 方法的时候需要指定 size，因为这个方法中的 view 是原生布局的 view，不是用 flex-box 来布局的，所以 CK 不能计算出这个 view 的高度，需要手动指定。

#### CKComponentViewClass

上面的例子中我们调用了 CKComponentViewClass 的其中一个构造方法：

```
CKComponentViewClass(Class viewClass)
```
调用这个方法，CK 会去调用对应 view 的指定的初始化方法，也就是 initWithFrame: 方法，但是如果某个 view 是我们自定义的，且重写了其指定初始化方法，那这个构造方法就不适合使用了，下面是 CKComponentViewClass 的另一个构造方法：

```
CKComponentViewClass(UIView *(*factory)(void))
```

此构造方法传入一个方法指针，这个方法返回对应的 View 实例对象，举个例子🌰：

```
static CustomView *createCustomView(void) {
    return [[CustomView alloc] initWithName:@"xxx"];
}
// ...
[CKComponent newWithView:{&createCustomView} size:{50, 50}];
```
✨注意看这个构造方法会发现传入的方法是不能传入任何参数的，如果这个自定义的 view 的初始化需要传入参数呢？下面说到的 CKViewComponentAttribute 就可以解决这个问题

#### CKViewComponentAttribute
在 例子1 中我们调用了 CKViewComponentAttribute 的其中一个构造方法：

```
CKComponentViewAttribute(SEL setter)
```
但是这个构造方法不能调用多个参数的方法，其另一个构造方法为我们提供了解决方法：

```
CKComponentViewAttribute(const std::string &ident, void (^app)(id view, id value))
```
此构造方法传入一个 identifier 和 一个 block，block 的两个参数分别是当前这个component 生成的 uiview 实例对象，第二个参数是我们传入的参数，在 block 中就可以直接调用原生的方法对 view 进行设置，请看例子2：

```
static const CKComponentViewAttribute buttonAttribute =
{"className.button.identifier", ^(UIButton *button, id value) {
    [button setImage:value forState:UIControlStateNormal];
}};

CKComponent *button = [CKComponent newWithView:{
   [UIButton class],
   {
       {@selector(setBackgroundColor:), [UIColor redColor]},
       {buttonAttribute, [UIImage imageNamed:@"xxx"]}
   }
} size:{.width = 30, .height = 30}];

```
buttonAttribute 中的参数 value 就是我们在下面传入的 [UIImage imageNamed:@"xxx"]

上面的写法可以简化一下，写得更紧凑些：

```
CKComponent *button = [CKComponent newWithView:{
   [UIButton class],
   {
       {@selector(setBackgroundColor:), [UIColor redColor]},
       {{"className.button.identifier", ^(UIButton *button, id value) {
           [button setImage:value forState:UIControlStateNormal];
       }}, [UIImage imageNamed:@"xxx"]}
   }
} size:{.width = 30, .height = 30}];
```

✨这里需要注意三点：
1. 需要保证所有使用了这个 view 的地方的 identifier 是唯一的，或者更保守一些，保证全局范围内是唯一的，这个 identifier 是 CK 是否重用 view 的其中一个关键的 key
2. 这里只能传入一个参数，如果要传多个参数，得自己封装一个
3. 重用的时候，如果传入和上一次设置的参数是一样的，block 不会被重复调用


## CKStatefulViewComponent & CKStatefulViewComponentController
CKStatefulViewComponent 及 controller 是提供了方法使得某个 cell 可以直接用原生代码实现，其中原生实现的 view 被称为 statefulView。
这个 component 与普通的 component 不同的地方有以下两点：
1. 同一个 statefulView 可以被移动在不同的 cell 中，普通的 component 只能复用
2. controller 可以延迟对 statefulView 的释放，防止 statefulView 进入重用池，也就是说 controller 可以保持 statefulView 的状态

那么 CKStatefulViewComponent 适用的场景是什么？看下面这个截图：

中间的推荐列表其实是在某个 cell 中放了一个 collectionview，而这个 collectionview 是需要保持其滑动位置的状态的（这个 cell 出屏幕后又入屏幕，collectionview 的滑动的位置还维持在上次用户浏览后的位置），这其实就是一个特别适合用 CKStatefulViewComponent 来实现的一个场景。

#### CKStatefulViewComponent
API 很简单，传入大小和辅助功能相关设置

```
+ (instancetype)newWithSize:(const CKComponentSize &)size
              accessibility:(const CKStatefulViewComponentAccessibility &)accessibility;
```
✨这里需要注意一点：
1. 对于 statefulView 是需要知道对应的大小的，因为 statefulView 是 UIView，内部的布局不是用 flex-box 实现的，CK 无法计算其高度，所有需要我们给定高度


#### CKStatefulViewComponentController
重点说说几个方法：

```
+ (UIView *)newStatefulView:(id)context;
```
需要重写的方法，返回 statefulView 的实例对象


```
+ (void)configureStatefulView:(UIView *)statefulView forComponent:(CKComponent *)component;
```
选择性重写的方法（根据需求决定是否重写），这是个利用给定的 component 的实例的 state 来配置所给的 view 的方法，其主要有两个目的：在 statefulView 出现在视图层级之前配置 view，和重用时重新配置当前 view


```
- (void)didAcquireStatefulView:(UIView *)statefulView NS_REQUIRES_SUPER;
```
选择性重写的方法，当 controller 得到 statefulView 后会调用，这个方法适合对 statefulView 进行一些设置，如果设置其 delegate 等


```
- (BOOL)canRelinquishStatefulView;
```
选择性重写的方法，返回 BOOL 值表示是否延迟释放 statefulView，返回 YES，statefulView 会被加入重用池中，返回 NO 则不加入直接释放


```
- (void)canRelinquishStatefulViewDidChange;
```
主动调用的方法，当上面 - (BOOL)canRelinquishStatefulView 方法的返回值有变化时需要调用这个方法，调用后 statefulView 可能会被加入重用池中。这个值什么时候会变化呢，比如说一个视频播放器，在全屏的时候不可释放，退出全屏可以释放



```
- (void)willRelinquishStatefulView:(UIView *)statefulView NS_REQUIRES_SUPER;
```
选择性重写的方法，在 statefulView 即将进入重用池前调用


```
+ (NSInteger)maximumPoolSize:(id)context;
```
选择性重写的方法，重用池的最大数量，不重写的话默认是-1，也就是没有限制



## 参考
http://componentkit.org/docs/views.html
http://componentkit.org/docs/advanced-views.html
http://componentkit.org/appledoc/html/Classes/CKStatefulViewComponentController.html
http://componentkit.org/appledoc/html/Classes/CKStatefulViewComponent.html

