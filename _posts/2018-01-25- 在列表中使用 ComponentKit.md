---
title: 在列表中使用 ComponentKit
---

### 关于 ComponentKit
`ComponentKit` 是 `Facebook` 发布的一个受 `React` 启发而产生的 `iOS` 视图框架，最开始用于 `Facebook` 的 `News Feed`，现在已经开始作为整个 `Facebook` 的 `iOS` 开发框架使用。
更多：[官方文档](http://componentkit.org/)、[一个中文介绍文档](https://segmentfault.com/a/1190000002625560)

-------


### 步骤
1. 实现 `CKComponentProvider` 协议
2. 为列表创建 `CKCollectionViewTransactionalDataSource`
3. 使用 `CKTransactionalComponentDataSourceChangeset`

### 实现 CKComponentProvider 协议
CKComponentProvider 只声明了一个方法：

```
+ (CKComponent *)componentForModel:(id<NSObject>)model context:(id<NSObject>)context;
```
这个方法需要我们将 `model` 转为成 `component`
通常我们将列表对应的 vc 作为这个协议的实现者。

✨ 为什么这个方法是个类方法，不是实例方法或是 `block`？
因为 `component` 是不可变的，为了保证这个方法返回的 `component` 不依赖其他的第三方属性，所以这里就将这个方法定义成了类方法。 

### 为列表创建 CKCollectionViewTransactionalDataSource
在创建之前先来看看 `CKDataSource` 是什么及作用
##### CKDataSource
* `CKDataSource` 在 CK 框架中起到非常核心的作用。
* `CKDataSource` 能够接收 `changesets`（后面会讲到），`changesets` 中包含 `commands` 和 `models`
* 根据所给的 `changeset` 在后台线程生成计算 `component`
* 将生成好的 `component` 输出给列表使用

##### CKCollectionViewTransactionalDataSource
`CKCollectionViewTransactionalDataSource` 是 `CKDataSource` 的一个简单的封装，在这个基础上又实现了 `UICollectionViewDataSource` 的 `API`，也就是说 CK 列表中的 `UICollectionViewDataSource` 不需要我们自己实现，CK 已经帮我们实现了。

##### 具体代码

```
// 将 component 的 size 范围设置成：宽度固定，高度自由变化，这个属性用于 component 高度的计算
CKComponentFlexibleSizeRangeProvider *sizeProvider =
[CKComponentFlexibleSizeRangeProvider providerWithFlexibility:CKComponentSizeRangeFlexibleHeight];
CKSizeRange sizeRange = [sizeProvider sizeRangeForBoundingSize:self.view.bounds.size];
    
// 生成 dataSource 需要的 configuration
CKTransactionalComponentDataSourceConfiguration *configuration =
[[CKTransactionalComponentDataSourceConfiguration alloc]
initWithComponentProvider:[self class]
context:self.context
sizeRange:sizeRange];

// 生成 dataSource
dataSource = [[CKCollectionViewTransactionalDataSource alloc] 
initWithCollectionView:self.collectionView 
supplementaryViewDataSource:nil 
configuration:configuration];
```

### 使用 CKTransactionalComponentDataSourceChangeset
##### changeset 是什么
`changeset` 是用来和 `datasource` 交流的一个工具，列表中所有的增、删、改操作都需要用 `changeset` 描述，然后再作用到 `datasource` 上。

##### API

```
/**
 Designated initializer. Any parameter may be nil.
 @param updatedItems Mapping from NSIndexPath to updated model.
 @param removedItems Set of NSIndexPath.
 @param removedSections NSIndexSet of section indices.
 @param movedItems Mapping from NSIndexPath to NSIndexPath.
 @param insertedSections NSIndexSet of section indices.
 @param insertedItems Mapping from NSIndexPath to new model.
 */
- (instancetype)initWithUpdatedItems:(NSDictionary<NSIndexPath *, ModelType> *)updatedItems
                        removedItems:(NSSet<NSIndexPath *> *)removedItems
                     removedSections:(NSIndexSet *)removedSections
                          movedItems:(NSDictionary<NSIndexPath *, NSIndexPath *> *)movedItems
                    insertedSections:(NSIndexSet *)insertedSections
                       insertedItems:(NSDictionary<NSIndexPath *, ModelType> *)insertedItems;
```
 在上面将 `CKDataSource` 的时候有提到过，changeset 包含 `commands` 和 `models`，从 `API` 中也可以体现出来，`update、remove、insert` 是 `commands`，后面的参数就是 `models`。
 但是这个 `API` 看起来很繁琐，需要传一大堆参数，下面这个 `ChangesetBuilder` 的 `API` 更加简洁：
  
```
CKTransactionalComponentDataSourceChangesetBuilder：
+ (instancetype)transactionalComponentDataSourceChangeset;
- (instancetype)withUpdatedItems:(NSDictionary<NSIndexPath *, ModelType> *)updatedItems;
- (instancetype)withRemovedItems:(NSSet *)removedItems;
- (instancetype)withRemovedSections:(NSIndexSet *)removedSections;
- (instancetype)withMovedItems:(NSDictionary<NSIndexPath *, NSIndexPath *> *)movedItems;
- (instancetype)withInsertedSections:(NSIndexSet *)insertedSections;
- (instancetype)withInsertedItems:(NSDictionary<NSIndexPath *, ModelType> *)insertedItems;
- (CKTransactionalComponentDataSourceChangeset<ModelType> *)build;
```

##### 具体代码

```
CKTransactionalComponentDataSourceChangesetBuilder *builder = [CKTransactionalComponentDataSourceChangesetBuilder new];
// 在 indexPath 位置 插入一个以 object 为 model 生成的 component
[builder withInsertedItems:@{indexPath: object}];
// 将 changeset 作用到 datasource 上
[self.dataSource applyChangeset:[builder build] mode:CKUpdateModeAsynchronous cellConfiguration:nil];
```
上面第二句中传的 `object` 会作为参数传入到 CKComponentProvider 方法中：

```
+ (CKComponent *)componentForModel:(id<NSObject>)model context:(id<NSObject>)context;
```

### 参考
http://componentkit.org/docs/datasource-overview.html
http://componentkit.org/docs/datasource-changeset-api.html

