---
title: Simple Ping in Swift - ICMP 报文和 IP 报文
---


## 引言
ICMP 是 IP 层协议，是为了提高 IP 数据交付成功率而使用的一种协议，是因特网标准协议（RFC 792）
ICMP 报文是作为 IP 层数据报的数据，加上 IP 数据报的首部，组成数据报发送出去，ICMP 是使用了 IP 协议的一种配套使用型协议

## ICMP
### 报文类型
* ICMP 报文分为两大类：ICMP 差错报告报文、ICMP 询问报文
* 不同细分类型由报文中的::类型::字段和::代码::字段来共同决定
具体细分类型如下：
![-w250](https://res.cloudinary.com/dp1pheuq7/image/upload/v1572098483/Simple_Ping_in_Swift_-_ICMP_%E6%8A%A5%E6%96%87%E5%92%8C_IP_%E6%8A%A5%E6%96%871_f0i9ch.jpg)

### 格式
一般格式：
![-w250](https://res.cloudinary.com/dp1pheuq7/image/upload/v1572098484/Simple_Ping_in_Swift_-_ICMP_%E6%8A%A5%E6%96%87%E5%92%8C_IP_%E6%8A%A5%E6%96%872_qcf6gd.png)
在 ping 的时候，使用的是 ICMP 回送请求或回答类型报文，对应的类型字段是 8 或者 0，对应的代码是 0
![-w250](https://res.cloudinary.com/dp1pheuq7/image/upload/v1572098412/Simple_Ping_in_Swift_-_ICMP_%E6%8A%A5%E6%96%87%E5%92%8C_IP_%E6%8A%A5%E6%96%873_ipyfw7.png)

### 代码
关于 Swift 的内存操作可以戳：[Simple Ping in Swift - 预备芝士 · YUI 的严肃文](https://linkexin.github.io/notes/Simple-Ping-in-Swift-%E9%A2%84%E5%A4%87%E8%8A%9D%E5%A3%AB)

定义一个 struct 用于表示 ICMP Header
* 各个字段的定义
```
public struct ICMPHeader {
    
    public static let size = 8
    public static let checksumDelta = 2
    
    // 类型
    public var type: UInt8 {
        didSet {
            headerBytes[0] = type
        }
    }
    // 代码
    public var code: UInt8 {
        didSet {
            headerBytes[1] = code
        }
    }
    // 校验和
    public var checksum: UInt16 {
        didSet {
            headerBytes[2...].withUnsafeMutableBytes {(bytes: UnsafeMutablePointer<UInt16>) in
                bytes.pointee = checksum.bigEndian
            }
        }
    }
    // 标识符
    public var identifier: UInt16 {
        didSet {
            headerBytes[4...].withUnsafeMutableBytes {(bytes: UnsafeMutablePointer<UInt16>) in
                bytes.pointee = identifier.bigEndian
            }
        }
    }
    // 序号
    public var sequenceNumber: UInt16 {
        didSet {
            headerBytes[6...].withUnsafeMutableBytes {(bytes: UnsafeMutablePointer<UInt16>) in
                bytes.pointee = sequenceNumber.bigEndian
            }
        }
    }
    
    public private(set) var headerBytes: Data
}
```

* 初始化方法：通过传入具体字段来生成 ICMP
```
// 通过传入具体字段来生成 ICMP
    public init(type t: UInt8, code c: UInt8, checksum chk: UInt16, identifier i: UInt16, sequenceNumber n: UInt16) {
        type = t
        code = c
        checksum = chk
        identifier = i
        sequenceNumber = n
        
        headerBytes = Data(count: ICMPHeader.size)
        // 使用指定的 UInt8 数据类型指针来访问 headerBytes 数据段
        headerBytes.withUnsafeMutableBytes {(bytes: UnsafeMutablePointer<UInt8>) in
            var curPosUInt8 = bytes
            curPosUInt8.pointee = type;
            curPosUInt8 = curPosUInt8.advanced(by: 1)
            
            curPosUInt8.pointee = code;
            curPosUInt8 = curPosUInt8.advanced(by: 1)
            
            var curPosUInt16 = UnsafeMutablePointer<UInt16>(OpaquePointer(curPosUInt8))
            curPosUInt16.pointee = checksum.bigEndian
            curPosUInt16 = curPosUInt16.advanced(by: 1)
            
            curPosUInt16.pointee = identifier.bigEndian
            curPosUInt16 = curPosUInt16.advanced(by: 1)
            
            curPosUInt16.pointee = sequenceNumber.bigEndian
            curPosUInt16 = curPosUInt16.advanced(by: 1)
        }
    }
```

* 初始化方法：通过传入的 data 来具体赋值各个字段
```
// 通过传入的 data 来具体赋值各个字段
    public init(data: Data) {
        assert(data.count >= ICMPHeader.size)
        
        var typeI: UInt8 = 0
        var codeI: UInt8 = 0
        var checksumI: UInt16 = 0
        var identifierI: UInt16 = 0
        var sequenceNumberI: UInt16 = 0
        data.withUnsafeBytes {(bytes: UnsafePointer<UInt8>) in
            var curPosUInt8 = bytes
            typeI = curPosUInt8.pointee;
            curPosUInt8 = curPosUInt8.advanced(by: 1)
            
            codeI = curPosUInt8.pointee;
            curPosUInt8 = curPosUInt8.advanced(by: 1)
            
            /* Note: UInt16(bigEndian:) <=> CFSwapInt16BigToHost() */
            var curPosUInt16 = UnsafePointer<UInt16>(OpaquePointer(curPosUInt8))
            // 按照 curPosUInt16.pointee 在存储器中存取的字节顺序来生成一个 UInt16 类型值
            checksumI = UInt16(bigEndian: curPosUInt16.pointee)
            curPosUInt16 = curPosUInt16.advanced(by: 1)
            
            identifierI = UInt16(bigEndian: curPosUInt16.pointee)
            curPosUInt16 = curPosUInt16.advanced(by: 1)
            
            sequenceNumberI = UInt16(bigEndian: curPosUInt16.pointee)
            curPosUInt16 = curPosUInt16.advanced(by: 1)
        }
        
        type = typeI
        code = codeI
        checksum = checksumI
        identifier = identifierI
        sequenceNumber = sequenceNumberI
        
        headerBytes = Data(data[..<ICMPHeader.size])
    }
```

## IP
### 数据报格式
![-w250](https://res.cloudinary.com/dp1pheuq7/image/upload/v1572098412/Simple_Ping_in_Swift_-_ICMP_%E6%8A%A5%E6%96%87%E5%92%8C_IP_%E6%8A%A5%E6%96%874_uzgakh.png)

### 代码
* 定义一个 struct 用来表示 IP 头部 4 字节地址
```
// IP 头部 4 字节地址
    struct Address {
        let byte1: UInt8
        let byte2: UInt8
        let byte3: UInt8
        let byte4: UInt8
        
        init() {
            byte1 = 0
            byte2 = 0
            byte3 = 0
            byte4 = 0
        }
        
        init(dataPointer: inout UnsafePointer<UInt8>) {
            // 逐字节赋值
            byte1 = dataPointer.pointee; dataPointer = dataPointer.advanced(by: 1)
            byte2 = dataPointer.pointee; dataPointer = dataPointer.advanced(by: 1)
            byte3 = dataPointer.pointee; dataPointer = dataPointer.advanced(by: 1)
            byte4 = dataPointer.pointee; dataPointer = dataPointer.advanced(by: 1)
        }
    }
```
* IP 数据报头部 struct
```
struct IPv4Header {
    static let size = 20
    
    // 版本（IPv4 or IPv6） + 首部长度
    let versionAndHeaderLength: UInt8
    // 服务类型，更低的延迟 or 更高的吞吐 or 更高的可靠性 or 代价更小的路由等等
    let differentiatedServices: UInt8
    // 总长度，首部和数据之和的长度，单位为字节
    let totalLength: UInt16
    // 标识，不是序号的意思，是为了分片后重装数据报
    let identification: UInt16
    // 标志 + 片偏移，都是分片相关参数
    let flagsAndFragmentOffset: UInt16
    // 生存时间，数据报在网络中可通过的路由器数的最大值
    let timeToLive: UInt8
    // 协议，数据报携带的z数据使用的是何种h协议
    let `protocol`: UInt8
    // 首部校验和
    let headerChecksum: UInt16
    // 原地址
    let sourceAddress: Address
    // 目的地址
    let destinationAddress: Address
    /* options... */
    /* data... */
    
    init(data: Data) {
        assert(data.count >= IPv4Header.size)
        
        var versionAndHeaderLengthI: UInt8 = 0
        var differentiatedServicesI: UInt8 = 0
        var totalLengthI: UInt16 = 0
        var identificationI: UInt16 = 0
        var flagsAndFragmentOffsetI: UInt16 = 0
        var timeToLiveI: UInt8 = 0
        var protocolI: UInt8 = 0
        var headerChecksumI: UInt16 = 0
        var sourceAddressI = Address()
        var destinationAddressI = Address()
        data.withUnsafeBytes { (bytes: UnsafePointer<UInt8>) in
            var curPosUInt8 = bytes
            versionAndHeaderLengthI = curPosUInt8.pointee
            curPosUInt8 = curPosUInt8.advanced(by: 1)
            
            differentiatedServicesI = curPosUInt8.pointee
            curPosUInt8 = curPosUInt8.advanced(by: 1)
            
            var curPosUInt16 = UnsafePointer<UInt16>(OpaquePointer(curPosUInt8))
            totalLengthI = curPosUInt16.pointee
            curPosUInt16 = curPosUInt16.advanced(by: 1)
            
            identificationI = curPosUInt16.pointee
            curPosUInt16 = curPosUInt16.advanced(by: 1)
            
            flagsAndFragmentOffsetI = curPosUInt16.pointee
            curPosUInt16 = curPosUInt16.advanced(by: 1)
            
            curPosUInt8 = UnsafePointer<UInt8>(OpaquePointer(curPosUInt16))
            timeToLiveI = curPosUInt8.pointee
            curPosUInt8 = curPosUInt8.advanced(by: 1)
            
            protocolI = curPosUInt8.pointee
            curPosUInt8 = curPosUInt8.advanced(by: 1)
            
            curPosUInt16 = UnsafePointer<UInt16>(OpaquePointer(curPosUInt8))
            headerChecksumI = curPosUInt16.pointee
            curPosUInt16 = curPosUInt16.advanced(by: 1)
            
            curPosUInt8 = UnsafePointer<UInt8>(OpaquePointer(curPosUInt16))
            sourceAddressI = Address(dataPointer: &curPosUInt8)
            destinationAddressI = Address(dataPointer: &curPosUInt8)
        }
        
        versionAndHeaderLength = versionAndHeaderLengthI
        differentiatedServices = differentiatedServicesI
        totalLength = totalLengthI
        identification = identificationI
        flagsAndFragmentOffset = flagsAndFragmentOffsetI
        timeToLive = timeToLiveI
        `protocol` = protocolI
        headerChecksum = headerChecksumI
        sourceAddress = sourceAddressI
        destinationAddress = destinationAddressI
    }
}
```

## IP 包 + ICMP包
将上面两个部分结合 ：
IP 首部 (20 字节) + 8 位类型 + 8 位代码 + 16 位校验和 + ICMP 首部其他部分 (7 字节) + 数据
![-w250](https://res.cloudinary.com/dp1pheuq7/image/upload/v1572098413/Simple_Ping_in_Swift_-_ICMP_%E6%8A%A5%E6%96%87%E5%92%8C_IP_%E6%8A%A5%E6%96%875_achhy1.png)
