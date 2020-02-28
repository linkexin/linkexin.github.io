---
title: AFSecurityPolicy
---

## 前言
分析 AF 的文章有特别多，但是关于 AFSecurityPolicy 这块，之前一直有些细节没有理解清楚，所以即使网上同类文章已经一大堆，但还是决定自己写写加深理解。

## 证书
首先需要了解，证书分为两种，一种是由 CA 认证签名的，另一种是自签名。
一种是花钱向认证的机构购买的证书，服务端如果使用的是这类证书的话，那一般客户端不需要做什么，用HTTPS进行请求就行了，因为操作系统中内置了那些受信任的根证书的。
另一种是自己制作的证书，怎么自己制作，也就是自己对自己签名，（其实根证书也是自己对自己进行签名）。使用这类证书默认是不受信任的（当然也不用花钱买），因此需要我们做一些额外的操作来完成验证：服务器产生一份自签名的证书，并且客户端内置一份服务器的证书，并将其设置为锚点证书。

自我签名的证书的信任链长度为1，也就是锚点证书就是自签名证书本身。

## 验证证书链
证书链的验证，主要由三部分来保证证书的可信：叶子证书是对应的请求的域名的证书，根证书是被系统信任的证书，以及这个证书链之间都是层层签发可信任链
证书之所以能成立，本质是基于信任链，这样任何一个节点证书加上域名校验，就确定一条唯一可信证书链。

## AF 的验证策略
AFSecurityPolicy 帮我们封装好了几种验证方式，总共有三种验证服务器是否被信任的方式：
```
typedef NS_ENUM(NSUInteger, AFSSLPinningMode) {
    AFSSLPinningModeNone,
    AFSSLPinningModeCertificate,
    AFSSLPinningModePublicKey,
};
```
* AFSSLPinningModeNone 是默认的认证方式，只会在系统的信任的证书列表中对服务端返回的证书进行验证，这种模式不需要导入证书，可以使用系统方法直接实现信任
* AFSSLPinningModeCertificate 需要客户端预先内置服务端的证书，在内置证书列表中对服务端返回的证书进行验证
* AFSSLPinningModePublicKey 也需要预先保存服务端发送的证书，但是这里只会验证证书中的公钥是否正确，有效期等其他信息忽略

### allowInvalidCertificates
表示是否允许无效证书。如果我们要验证自签名证书，就需要将这个置为 YES，表示需要验证无效证书（自签名证书）

### validatesDomainName
表示是否验证域名。假如证书上注册的域名是 www.google.com，那么 mail.google.com 是无法验证通过的（可以通过注册通配符的域名 *.google.com 证书，但这个还是比较贵的）

## evaluateServerTrust
首先来看类中最重要的方法 `- (BOOL)evaluateServerTrust:(SecTrustRef)serverTrust forDomain:(NSString *)domain`
参数 SecTrustRef 是信任管理对象，里面包含着服务器返回的证书链，此对象贯穿着验证的整个过程，需要验证的 domain、策略等等信息都需要设置到这个对象中，最后将此对象传递给系统方法，系统根据设定进行验证

整个方法的大概可以分为几个部分：
```
- (BOOL)evaluateServerTrust:(SecTrustRef)serverTrust
                  forDomain:(NSString *)domain {
    ① 判断 设置的策略 是否合法
    ② 设置验证证书的策略
    ③ AFSSLPinningModeNone 方式的验证
    ④ AFSSLPinningModeCertificate 方式的验证
    ⑤ AFSSLPinningModePublicKey 方式的验证
}
```

### ① 判断 设置的策略 是否合法
```
// 如果想要验证自签名证书的域名是否有效，那么本地必须有自签名证书。
// 而 AFSSLPinningModeNone 表示只验证系统的证书链；[self.pinnedCertificates count] == 0 表示本地没有自签名证书，都是不允许的，直接验证不通过
if (domain && self.allowInvalidCertificates && self.validatesDomainName && (self.SSLPinningMode == AFSSLPinningModeNone || [self.pinnedCertificates count] == 0)) {
    // https://developer.apple.com/library/mac/documentation/NetworkingInternet/Conceptual/NetworkingTopics/Articles/OverridingSSLChainValidationCorrectly.html
    //  According to the docs, you should only trust your provided certs for evaluation.
    //  Pinned certificates are added to the trust. Without pinned certificates,
    //  there is nothing to evaluate against.
    //
    //  From Apple Docs:
    //          “Do not implicitly trust self-signed certificates as anchors (kSecTrustOptionImplicitAnchors).
    //           Instead, add your own (self-signed) CA certificate to the list of trusted anchors.”
    NSLog(@“In order to validate a domain name for self signed certificates, you MUST use pinning.”);
    return NO;
}
```

### ② 设置验证证书的策略
```
// 根据条件创建验证证书的策略
NSMutableArray *policies = [NSMutableArray array];
if (self.validatesDomainName) {
    // 如果需要验证域名，传入特定参数创建一个验证策略的对象
    // 第一个参数表示需要验证整个证书链，第二个参数传入 domain 用与于验证叶子证书的 domain 和此 domain 是否一致
    // 会返回一个 SecPolicyRef 表示一个用于计算 SSL 证书链的 policy 对象
    // SecPolicyRef 是需要调用者手动管理内存的，AF 这里通过桥接将对象的管理权交给了 ARC
    [policies addObject:(__bridge_transfer id)SecPolicyCreateSSL(true, (__bridge CFStringRef)domain)];
} else {
    // 如果不需要验证域名，就使用 X.509，现有密码学里公钥证书的格式标准
    // X.509 策略不验证域名，返回的服务器证书，只要是可信任 CA 机构签发的，都会校验通过
    [policies addObject:(__bridge_transfer id)SecPolicyCreateBasicX509()];
}

// 设置验证证书的策略，所有验证需要的参数，都要往 serverTrust 中设置
SecTrustSetPolicies(serverTrust, (__bridge CFArrayRef)policies);
```

### ③ AFSSLPinningModeNone 方式的验证
```
// AFSSLPinningModeNone 表示只在系统的信任的证书列表中对服务端返回的证书进行验证
if (self.SSLPinningMode == AFSSLPinningModeNone) {
    // 如果允许不合法的证书，那直接返回 true
    // 或者直接调用系统方法验证此证书在现有的证书列表中（现在只有系统中自带的证书）是否能被正确验证
    return self.allowInvalidCertificates || AFServerTrustIsValid(serverTrust);
} else if (!AFServerTrustIsValid(serverTrust) && !self.allowInvalidCertificates) {
    // 如果验证不过且不允许自签名证书，那直接判断验证不过
    return NO;
}
```

### ④ AFSSLPinningModeCertificate 方式的验证
```
 case AFSSLPinningModeCertificate: {
    NSMutableArray *pinnedCertificates = [NSMutableArray array];
    for (NSData *certificateData in self.pinnedCertificates) {
        // 将客户端的证书添加到数组中，SecCertificateCreateWithData 返回一个代表「用 DER 编码的证书」的对象
        [pinnedCertificates addObject:(__bridge_transfer id)SecCertificateCreateWithData(NULL, (__bridge CFDataRef)certificateData)];
    }
    // 将客户端的证书设置为锚点证书，锚点证书一般是那些嵌入到操作系统中的根证书，顺着证书链层层验证知道锚点证书，那验证就成功
    // 如果我们使用自签名证书，那就是不由 CA 机构签发的，也就不能通过操作系统中的根证书验证，所以我们要将客户端的证书设置为锚点证书，才可能完成自签名证书的验证
    SecTrustSetAnchorCertificates(serverTrust, (__bridge CFArrayRef)pinnedCertificates);

    if (!AFServerTrustIsValid(serverTrust)) {
        return NO;
    }

    // 下面这一段之前一直疑惑，上面 AFServerTrustIsValid 不是已经验证过了吗，那这里是在做什么？
    // 其实 AFServerTrustIsValid 方法中只是通过验证其签名以及证书链中证书的签名（直到锚证书）来验证证书
    // 而这里是将整个证书进行验证，包括证书中的其他信息，比如说有效期等、
    // 下面的 AFSSLPinningModePublicKey 方法中，因为只验证公钥是否正确，所以就不包含这部分逻辑
    
    // 服务器端返回的证书链，注意此处返回的证书链顺序是从叶节点到根节点
    // obtain the chain after being validated, which *should* contain the pinned certificate in the last position (if it’s the Root CA)
    NSArray *serverCertificates = AFCertificateTrustChainForServerTrust(serverTrust);
    
    // 如果服务端返回的证书链中包含客户端本地的证书，说明服务器端是可信的
    for (NSData *trustChainCertificate in [serverCertificates reverseObjectEnumerator]) {
        if ([self.pinnedCertificates containsObject:trustChainCertificate]) {
            return YES;
        }
    }
    return NO;
}
```

### ⑤ AFSSLPinningModePublicKey 方式的验证
```
// 只验证时只验证证书里的公钥，不验证证书的有效期等信息
case AFSSLPinningModePublicKey: {
    NSUInteger trustedPublicKeyCount = 0;
    // 服务端证书中的所有公钥
    NSArray *publicKeys = AFPublicKeyTrustChainForServerTrust(serverTrust);

    // 看看是否存在客户端证书的公钥和服务端证书公钥一致的情况
    for (id trustChainPublicKey in publicKeys) {
        for (id pinnedPublicKey in self.pinnedPublicKeys) {
            if (AFSecKeyIsEqualToKey((__bridge SecKeyRef)trustChainPublicKey, (__bridge SecKeyRef)pinnedPublicKey)) {
                trustedPublicKeyCount += 1;
            }
        }
    }
    return trustedPublicKeyCount > 0;
}
```

## AFServerTrustIsValid
传入设置好的 SecTrustRef，并调用系统方法进行验证
```
static BOOL AFServerTrustIsValid(SecTrustRef serverTrust) {
    BOOL isValid = NO;
    SecTrustResultType result;
    // SecTrustEvaluate 根据信任管理对象中包含的一个或多个策略，通过验证其签名以及证书链中证书的签名（直到锚证书）来验证证书。
    // SecTrustEvaluate 其实是一个同步方法，__Require_noErr_Quiet 这个宏的作用就是一直循环着等待 result 有结果
    // 如果验证失败，那就直接通过 goto 语句跳到 _out 处返回失败结果
    __Require_noErr_Quiet(SecTrustEvaluate(serverTrust, &result), _out);
    
    // 以下两种结果表示验证通过
    isValid = (result == kSecTrustResultUnspecified || result == kSecTrustResultProceed);

_out:
    return isValid;
}
```

## 还有一点点
* SecTrustEvaluate 在 iOS 13 被 deprecated 了，取而代之的是 SecTrustEvaluateWithError，新方法直接返回一个 bool 值，只要判断返回值的 false true 就可以了，比旧 API 更简单友好
* 其实整个流程理解清楚了以后，会发现 AFSecurityPolicy 做的事情是根据设置将参数（domain、策略）设置到 SecTrustRef 中，最后再将 SecTrustRef 传递给系统方法，执行验证过程。
而一开始的 SecTrustRef 是从 NSURLSessionTaskDelegate 的回调方法中传递过来的：

```
- (void)URLSession:(NSURLSession *)session
              task:(NSURLSessionTask *)task
didReceiveChallenge:(NSURLAuthenticationChallenge *)challenge
 completionHandler:(void (^)(NSURLSessionAuthChallengeDisposition disposition, NSURLCredential *credential))completionHandler {
    [self.securityPolicy evaluateServerTrust:challenge.protectionSpace.serverTrust forDomain:host];
}
```
