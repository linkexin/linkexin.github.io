# 系统分享 Share Extension

##说在前面

扩展（ Extension ）是 iOS 8 中引入的一个非常重要的新特性。扩展让 app 之间的数据交互成为可能。用户可以在 app 中使用其他应用提供的功能，而无需离开当前的应用。

其实扩展并不是 iOS 8 才有的，在早先版本里，从 iOS5 到 iOS7，分享插件的界面设计经过了几次变更，但是功能上一直十分有限，一开始仅限于系统级和系统原生应用的内容传递。后来苹果通过与Twitter和Facebook等几家公司签订独立的协议，实现了整合的方案，使内容分享到这些应用的过程更方便。
![](http://res.cloudinary.com/dp1pheuq7/image/upload/v1513092675/ShareExtension1_nrgune.png)

##基本介绍

####Containing App

扩展并不是一个独立的 app，扩展的 bundle 必须包含在一个普通应用的 bundle 的内部，这就意味着拓展不能单独存在，必须有一个包含它的 Containing App

####Host App

扩展需要有应用来调用其运行，调用扩展的应用称为 Host App

####创建

![](http://res.cloudinary.com/dp1pheuq7/image/upload/v1513092672/ShareExtension2_l8htrv.png)

目录结构：
![](http://res.cloudinary.com/dp1pheuq7/image/upload/v1513092672/ShareExtension3_rqpmb1.png)

run 起来以后：
![](http://res.cloudinary.com/dp1pheuq7/image/upload/v1513092675/ShareExtension4_frypmo.png)


####父类 SLComposeServiceViewController

提供了一些方法：

```
- (BOOL)isContentValid;
决定是否可以点击 Post 按钮
 
- (UIView *)loadPreviewView
加载预览视图
 
- (void)didSelectPost;
点击 Post 按钮时调用
 
- (void)didSelectCancel;
点击 Cancel 按钮时调用
 
- (NSArray *)configurationItems;
展示页下面可以点选的 item，返回一组包含 SLComposeSheetConfigurationItem 类型对象的数组
 
- (void)reloadConfigurationItems;
刷新一遍可点选的 item
 
- (void)presentationAnimationDidFinish;
展示完成时被调用，生命周期比较早，一般用来执行预加载任务
```

####共享数据
![](http://res.cloudinary.com/dp1pheuq7/image/upload/v1513092672/ShareExtension5_p7emjg.jpg)

![](http://res.cloudinary.com/dp1pheuq7/image/upload/v1513092671/ShareExtension6_yp1tjd.jpg)

####与 Containing App 共享数据
通过 App Group 实现 Extension 和 Containing App 数据的共享
![](http://res.cloudinary.com/dp1pheuq7/image/upload/v1513092673/ShareExtension7_kkowqi.jpg)

entitlements 文件自动生成关于 App Group 的相关配置
![](http://res.cloudinary.com/dp1pheuq7/image/upload/v1513092673/ShareExtension8_jbt2ig.jpg)

代码实现

具体的实现方式可以是：NSUserDefault、NSFileManager、CoreData

* NSUserDefault

```
NSUserDefaults *groupUserDefault = [[NSUserDefaults alloc] initWithSuiteName:@“group.com.zhihu.ios”];
[groupUserDefault setObject:password forKey:key];
[groupUserDefault synchronize];
```

* NSFileManager
通过调用 containerURLForSecurityApplicationGroupIdentifier:方法可以获得 AppGroup 的共享目录

```
NSURL *groupURL = [[NSFileManager defaultManager] containerURLForSecurityApplicationGroupIdentifier:@“group.com.zhihu.ios”];
NSURL *fileURL = [groupURL URLByAppendingPathComponent:@"demo.txt"];
//写入文件
[@"abc" writeToURL:fileURL atomically:YES encoding:NSUTF8StringEncoding error:nil];
//读取文件
NSString *str = [NSString stringWithContentsOfURL:fileURL encoding:NSUTF8StringEncoding error:nil];
NSLog(@"str = %@", str);
```

* CoreData

```
//获取分组的共享项目
NSURL *containerURL = [[NSFileManager defaultManager] containerURLForSecurityApplicationGroupIdentifier:@“group.com.zhihu.ios”];
NSURL *storeURL = [containerURL URLByAppendingPathComponent:@"DataModel.sqlite"];
 
//初始化持久化存储调度器
NSURL *modelURL = [[NSBundle mainBundle] URLForResource:@"DataModel" withExtension:@"momd"];
 
NSManagedObjectModel *model = [[NSManagedObjectModel alloc] initWithContentsOfURL:modelURL];
NSPersistentStoreCoordinator *coordinator = [[NSPersistentStoreCoordinator alloc] initWithManagedObjectModel:model];
 
[coordinator addPersistentStoreWithType:NSSQLiteStoreType
configuration:nil
URL:storeURL
options:nil
error:nil];
 
//创建受控对象上下文
NSManagedObjectContext *context = [[NSManagedObjectContext alloc] initWithConcurrencyType:NSMainQueueConcurrencyType];
 
[context performBlockAndWait:^{
[context setPersistentStoreCoordinator:coordinator];
}];
```

####与 Host App 共享数据
![](http://res.cloudinary.com/dp1pheuq7/image/upload/v1513092675/ShareExtension9_pavyco.png)

代码实现

```
for (NSExtensionItem *item in self.extensionContext.inputItems) {
    for (NSItemProvider *itemProvider in item.attachments) {
        if ([itemProvider hasItemConformingToTypeIdentifier:(NSString       *)kUTTypeURL]) {
            [itemProvider loadItemForTypeIdentifier:(NSString *)kUTTypeURL
                                            options:nil
                                  completionHandler:^(id
                                                      _Nullable item,
                                                      NSError * _Null_unspecified error) {
                                      if ([(NSObject *)item isKindOfClass:[NSURL class]]) {
                                          NSLog(@"分享的URL = %@", item);
                                      }
                                  }];
        }
    }
}
```

####共享 Keychain 数据
* 开启 Keychain Group
![](http://res.cloudinary.com/dp1pheuq7/image/upload/v1513092673/ShareExtension10_w4ccus.jpg)

* entitlements 文件自动生成关于 Keychain Group 的相关配置
![](http://res.cloudinary.com/dp1pheuq7/image/upload/v1513092674/ShareExtension11_vgdz9b.jpg)


* 在构造请求参数的时候必须加上 Keychain Group

```
存：
NSMutableDictionary *query = [NSMutableDictionary new];
[query setObject:service forKey:(__bridge id)kSecAttrService];
[query setObject:account forKey:(__bridge id)kSecAttrAccount];
[query setObject:self.passwordData forKey:(__bridge id)kSecValueData];
[query setObject:keychainGroupName forKey:(__bridge id)kSecAttrAccessGroup];
SecItemAdd((__bridge CFDictionaryRef)query, NULL);
 
取：
CFTypeRef result = NULL;
NSMutableDictionary *query = [NSMutableDictionary new];
[query setObject:service forKey:(__bridge id)kSecAttrService];
[query setObject:account forKey:(__bridge id)kSecAttrAccount];
[query setObject:keychainGroupName forKey:(__bridge id)kSecAttrAccessGroup];
SecItemCopyMatching((__bridge CFDictionaryRef)query, &amp;result);
self.passwordData = (__bridge_transfer NSData *)result;
```

####注意事项
* 结束扩展程序，必须要调用

```
[self.extensionContext completeRequestReturningItems:@[] completionHandler:nil];
```

* 不要使用与主 app 信息相关的类，例如：UIApplication，不能使用的在 API 中都会用宏 NS_EXTENSION_UNAVAILABLE 标明

* 配置激活扩展的规则 NSExtensionActivationRule，默认是 TRUEPREDICATE

```
NSExtensionActivationSupportsAttachmentsWithMaxCount
附件最多限制，为数值类型。附件包括File、Image和Movie三大类，单一、混选总量不超过指定数量
 
NSExtensionActivationSupportsAttachmentsWithMinCount
附件最少限制，为数值类型。
 
NSExtensionActivationSupportsFileWithMaxCount
文件最多限制，为数值类型。文件泛指除Image/Movie之外的附件，例如【邮件】附件、【语音备忘录】等。
单一、混选均不超过指定数量。
 
NSExtensionActivationSupportsImageWithMaxCount
图片最多限制，为数值类型。
 
NSExtensionActivationSupportsMovieWithMaxCount
视频最多限制，为数值类型。
```
或者可以使用谓词来描述

```
NSExtensionActivationRule
SUBQUERY (
extensionItems,
$extensionItem,
SUBQUERY (
$extensionItem.attachments,
$attachment,
(
ANY $attachment.registeredTypeIdentifiers UTI-CONFORMS-TO "public.url"
)
).@count &gt;= 1
).@count == 1
```

