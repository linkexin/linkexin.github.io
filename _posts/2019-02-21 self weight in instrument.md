---
title: self weight in instrument
---

### 问题
我们知道 weight 表示执行某方法消耗的时间，但是一直不是很明确 self weight 的具体含义，之前查过一些资料：
[objective c - Profiling with Instruments : Whats the difference b/w self weight and weight - Stack Overflow](https://stackoverflow.com/questions/46314441/profiling-with-instruments-whats-the-difference-b-w-self-weight-and-weight)
![-w250](https://res.cloudinary.com/dp1pheuq7/image/upload/v1550861261/716BB653-A452-4FD1-8CB3-E39D3AB5763C_1_pbgs3l.jpg)
里面提到「Self weight is the weight without the time in the children」但是我一直不明确这里的「children」是什么意思，所以还是决定自己试一下

### 实验
记得将「hide system libraries」打开，这样看更明确
首先创建两个类 superclass 和 subclass，superclass 内部只调用系统方法
```
@implementation superclass
- (instancetype)init {
    if (self = [super init]) {
        for (long i = 0; i < 10000000000; i ++) {
            NSLog(@"12312321");
        }
    }
    return self;
}
@end

@implementation subclass
- (instancetype)init {
    if (self = [super init]) {
        
    }
    return self;
}
@end
```

得到以下数据：
![-w250](https://res.cloudinary.com/dp1pheuq7/image/upload/v1550861259/%E4%BC%81%E4%B8%9A%E5%BE%AE%E4%BF%A1%E6%88%AA%E5%9B%BE_f87d9606-082c-445f-97e3-c7eea94bf993_qffeie.png)
可以看到 [superclass init] 方法的 self weight 值基本与 weight 的值一致了，说明在勾选了「hide system libraries」的情况下，self weight 表示的是这个方法本身消耗的时间，也就是方法内部所有的系统方法所消耗的时间。

为了进一步验证这个结论，将 superclass 的实现改一下，在 init 方法中写了一段耗时的操作，且特地调用了自定义的方法
```
@implementation superclass
- (instancetype)init {
    if (self = [super init]) {
        for (long i = 0; i < 10000000000; i ++) {
            [self keychain];
        }
    }
    return self;
}

- (void)keychain {
    [SAMKeychain passwordForService:@"xin_keychain_test" account:@"xin"];
}
@end
```

得到数据：
![-w250](https://res.cloudinary.com/dp1pheuq7/image/upload/v1550861266/%E4%BC%81%E4%B8%9A%E5%BE%AE%E4%BF%A1%E6%88%AA%E5%9B%BE_68791ecf-c8ce-4206-9591-4540f72777e7_esvvsx.png)
可以看到 [superclass init] 方法的 self weight 值大大减小了，相应的 [SAMKeychainQuery fetch:] 的 self weight 值很大，因为此方法内部耗时的地方大部分在系统的调用方法上了，所以基本上可以印证我们上面的结论

### 结论
Self weight 表示的是这个方法本身消耗的时间，也就是方法内部所有的系统方法所消耗的时间。
回到一开始提到的「Self weight is the weight without the time in the children」，个人认为这里的 children 应该是方法内部调用的非系统方法

