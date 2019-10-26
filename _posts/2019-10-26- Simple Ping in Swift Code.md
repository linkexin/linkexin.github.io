---
title: Simple Ping in Swift Code
---


文中代码主要是参考：[GitHub - Frizlab/SimpleSwiftPing: A Swift implementation of SimplePing from the Apple Developer Examples](https://github.com/Frizlab/SimpleSwiftPing)

## 前言
iOS 平台上想使用 ping 不像安卓可以直接使用 linux 的 ping 命令，我们需要自己来实现，早先苹果给出了一个  OC 版本的 demo，并参考了网上一些 Swift 版本的实现，虽然只是个小工具，但还是涉及到了不少知识点：ICMP 报文格式、Swift 中的指针操作、底层 C 方法等，稍作整理。
前三个部分可以戳：
[Size, Stride, Alignment · YUI 的严肃文](https://linkexin.github.io/notes/Size,-Stride,-Alignment)
[Simple Ping in Swift - 预备芝士 · YUI 的严肃文](https://linkexin.github.io/notes/Simple-Ping-in-Swift-%E9%A2%84%E5%A4%87%E8%8A%9D%E5%A3%AB)
[Simple Ping in Swift - ICMP 报文和 IP 报文 · YUI 的严肃文](https://linkexin.github.io/notes/Simple-Ping-in-Swift-ICMP-%E6%8A%A5%E6%96%87%E5%92%8C-IP-%E6%8A%A5%E6%96%87)

## Swift 代码实现 
### ICMP 和 IP 结构体
### 初始化
```
// 需要 ping 的域名
public let hostName: String
// 需要在开始 ping 之前指名
public var addressStyle: AddressStyle
// 用来标识一个 ping，init 的时候会赋值一个随机数
public let identifier: UInt16

public init(hostName hn: String, addressStyle s: AddressStyle = .any) {
    hostName = hn
    addressStyle = s
    identifier = UInt16.random(in: .min ... .max)
}
```
初始化的时候需要调用者传递一些必要信息，并生成一个随机数作为标识符

### DNS 解析
```
fileprivate var host: CFHost?
public private(set) var hostAddress: Data?

public func start() {
    assert(host == nil)
    assert(hostAddress == nil)
    
    // 在创建 context 的时候将 self 本身作为 info 参数传递进去，在回调函数中会使用到
    var context = CFHostClientContext(version: 0,
                                      info: unsafeBitCast(self, to: UnsafeMutableRawPointer.self),
                                      retain: nil,
                                      release: nil,
                                      copyDescription: nil)
    
    // 使用现有的 hostName（非 IP） 创建一个 CFHostRef 对象
    let h = CFHostCreateWithName(nil, hostName as CFString).autorelease().takeUnretainedValue()
    host = h
    // 提供一个上下文对象和回调函数
    CFHostSetClient(h, hostResolveCallback, &context)
    // 在 RunLoop 中执行具体的解析操作
    CFHostScheduleWithRunLoop(h, CFRunLoopGetCurrent(), RunLoopMode.defaultRunLoopMode.rawValue as CFString)
    
    var error = CFStreamError()
    // 开始解析，把它的第二个参数设置为 .addresses 表明你想要返回一个 IP 地址
    if !CFHostStartInfoResolution(h, CFHostInfoType.addresses, &error) {
        didFail(hostStreamError: error)
    } else {
        //set a timeout behavior
        setTimeoutBehaviorOfPing()
    }
}

// 回调方法
private func hostResolveCallback(theHost: CFHost,
                                 typeInfo: CFHostInfoType,
                                 error: UnsafePointer<CFStreamError>?,
                                 info: UnsafeMutableRawPointer?) {
    /* This C routine is called by CFHost when the host resolution is complete.
       * It just redirects the call to the appropriate Swift method. */
    // 装换为 Swift Object
    let obj = unsafeBitCast(info, to: SimplePing.self)
    // 如果正确查询到结果，那么 host 此时已经被赋值，因为在 CFHostSetClient 的时候已经将 .host 作为参数传递进去了
    assert(obj.host === theHost)
    assert(typeInfo == CFHostInfoType.addresses)
    
    if let error = error, error.pointee.domain != 0 {
        obj.didFail(hostStreamError: error.pointee)
    } else {
        obj.hostResolutionDone()
    }
}

// 获取到 IP 组以后我们需要解析出一个符合要求的可用的 IP
fileprivate func hostResolutionDone() {
    // DarwinBoolean is the Swift mapping of the "historic" C type Boolean
    var resolved = DarwinBoolean(false)
    guard let h = host else {
        return
    }
    // 函数来获取解析结果，这个函数返回一个数组，因为一个域名可能会对应多个 IP，我们只选取第一个可用的 IP 即可
    let addresses = CFHostGetAddressing(h, &resolved)?.retain().autorelease()
    if resolved.boolValue, let addresses = addresses?.takeUnretainedValue() as? [Data] {
        resolved = false
        for address in addresses {
            assert(hostAddress == nil)
            guard address.count >= MemoryLayout<sockaddr>.size else {
                continue
            }
            
            address.withUnsafeBytes {(addrPtr: UnsafePointer<sockaddr>) in
                // 根据初始化传入的 addressStyle 进行判断
                switch (addrPtr.pointee.sa_family, addressStyle) {
                case (sa_family_t(AF_INET), .any), (sa_family_t(AF_INET), .icmpV4):
                    hostAddress = address; resolved = true
                case (sa_family_t(AF_INET6), .any), (sa_family_t(AF_INET6), .icmpV6):
                    hostAddress = address; resolved = true
                default: ()
                }
            }
            if resolved.boolValue {
                break
            }
        }
    }
    
    stopHostResolution()
    
    if resolved.boolValue {
        assert(hostAddress != nil)
        startWithHostAddress()
    } else {
        didFail(error: NSError(domain: kCFErrorDomainCFNetwork as String,
                               code: Int(CFNetworkErrors.cfHostErrorHostNotFound.rawValue),
                               userInfo: nil))
    }
}
```

### 套接字创建
域名解析成功后，我们需要创建套接字，为后续发送包做准备
```
fileprivate var sock: CFSocket?

private func startWithHostAddress() {
    // Type for the platform-specific native socket handle.
    let fd: CFSocketNativeHandle
    let err: Int32
    switch hostAddressFamily {
    case sa_family_t(AF_INET):
        // SOCK_DGRAM 表示使用不可靠的数据报服务，不保障有序、可达
        fd = socket(AF_INET, SOCK_DGRAM, IPPROTO_ICMP)
        if fd < 0 {
            err = errno
        } else {
            err = 0
        }
        
    case sa_family_t(AF_INET6):
        fd = socket(AF_INET6, SOCK_DGRAM, IPPROTO_ICMPV6)
        if fd < 0 {
            err = errno
        } else {
            err = 0
        }
        
    default:
        fd = -1
        err = EPROTONOSUPPORT
    }
    
    guard err == 0 else {
        didFail(error: NSError(domain: NSPOSIXErrorDomain, code: Int(err), userInfo: nil))
        return
    }
    
    // 包装到 CFSocket 中，仍然将 self 本身作为 info 参数传递
    var context = CFSocketContext(version: 0,
                                  info: unsafeBitCast(self, to: UnsafeMutableRawPointer.self),
                                  retain: nil,
                                  release: nil,
                                  copyDescription: nil)
    // CFSocketCreateWithNative 会返回一个可复用的 socket
    // readCallBack: The callback is called when data is available to be read or a new connection is waiting to be accepted. The data is not automatically read; the callback must read the data itself.
    sock = CFSocketCreateWithNative(nil, fd, CFSocketCallBackType.readCallBack.rawValue, socketReadCallback, &context)
    assert(sock != nil)
    
    assert(CFSocketGetSocketFlags(sock) & kCFSocketCloseOnInvalidate != 0)
    // 为 CFSocket 创建一个 CFRunLoopSourceRef
    let rls = CFSocketCreateRunLoopSource(nil, sock, 0)
    assert(rls != nil)
    
    CFRunLoopAddSource(CFRunLoopGetCurrent(), rls, CFRunLoopMode.defaultMode)
    delegate?.simplePing(self, didStart: hostAddress ?? Data())
}
```

### 发送消息
因为 ping 默认使用  UDP，是无连接的，所以发送数据的时候需要致命对方的 IP 地址
构造 ICMP 包，ICMPHeader 的定义可以查看：
```
func pingPacket(type: UInt8, payload: Data, requiresChecksum: Bool) -> Data {
        let header = ICMPHeader(
            type: type, code: 0, checksum: 0,
            identifier: identifier, sequenceNumber: nextSequenceNumber
        )
        
        var packet = header.headerBytes + payload
        if requiresChecksum {
            /* The IP checksum routine returns a 16-bit number that's already in
               * correct byte order (due to wacky 1's complement maths), so we just
               * put it into the packet as a 16-bit unit. */
            // IP 只针对头部进行校验，ICMP 是基于整个包来生成校验和
            let checksumBig = SimplePing.packetChecksum(packetData: packet)
            packet[ICMPHeader.checksumDelta...].withUnsafeMutableBytes {(bytes: UnsafeMutablePointer<UInt16>) in bytes.pointee = checksumBig }
        }
        
        return packet
    }
```

检验和的生成：
```
static private func packetChecksum(packetData: Data) -> UInt16 {
        var sum: Int32 = 0
        var packetData = packetData
        
        /* Mop up an odd byte, if necessary */
        if packetData.count % 2 == 1 {
            packetData += Data([0])
        }
        
        /* Our algorithm is simple, using a 32 bit accumulator (sum), we
           * add sequential 16 bit words to it, and at the end, fold back all the
           * carry bits from the top 16 bits into the lower 16 bits. */
        packetData.withUnsafeBytes {(bytes: UnsafePointer<UInt16>) in
            var curPos = bytes
            assert(packetData.count % 2 == 0)
            for i in 0 ..< packetData.count / 2 {
                //跳过校验和区域，因为我们正在计算校验和
                if i != ICMPHeader.checksumDelta / 2 {
                    // 从低位到高位，按照 16bit 为单位相加
                    sum &+= Int32(curPos.pointee)
                }
                curPos = curPos.advanced(by: 1)
            }
        }
        
        /* Add back carry outs from top 16 bits to low 16 bits */
        sum = (sum >> 16) &+ (sum & 0xffff)            /* add high 16 to low 16 */
        sum &+= (sum >> 16)                              /* add carry */
        return UInt16(truncating: NSNumber(value: ~sum)) /* truncate to 16 bits */
    }
```

### 发送包
构造好包，调用 sendto 方法来发送
```
public func sendPing(data: Data?) {
    // 必须要等到 IP 地址解析完成才能开始 ping
    guard let hostAddress = hostAddress else {
        fatalError("Gotta wait for -simplePing:didStartWithAddress: before sending a ping")
    }
    
    /* Our dummy payload is sized so that the resulting ICMP packet, including
       * the ICMPHeader, is 64-bytes, which makes it easier to recognise our
       * packets on the wire. */
    let payload = data ?? String(format: "%28zd bottles of beer on the wall", 99 - (nextSequenceNumber % 100)).data(using: .ascii)!
    assert(data != nil || payload.count == 56)// 56 为 64 扣掉首部的长度
    
    // 构造 ping packet
    let packet: Data
    switch hostAddressFamily {
    case sa_family_t(AF_INET):
        packet = pingPacket(type: ICMPv4TypeEcho.request.rawValue, payload: payload, requiresChecksum: true)
    case sa_family_t(AF_INET6):
        packet = pingPacket(type: ICMPv6TypeEcho.request.rawValue, payload: payload, requiresChecksum: true)
    default:
        fatalError()
    }
    
    // 调用 sendto 方法发送 package
    let err: Int32
    let bytesSent: Int
    if let socket = sock {
        bytesSent = packet.withUnsafeBytes {(packetBytes: UnsafePointer<UInt8>) -> Int in
            return hostAddress.withUnsafeBytes {(hostAddressBytes: UnsafePointer<sockaddr>) -> Int in
                return sendto(
                    CFSocketGetNative(socket),
                    UnsafeRawPointer(packetBytes),
                    packet.count,
                    0, /* flags */
                    hostAddressBytes, //因为是基于 UDP 发送，所以是无连接的，需要致命目的地址
                    socklen_t(hostAddress.count)
                )
            }
        }
        // 如果发送成功，sendto 方法会返回实际传送出去的字符长度，失败返回 -1
        if bytesSent >= 0 {
            err = 0
        } else {
            err = errno
        }
    } else {
        bytesSent = -1
        err = EBADF
    }
    
    if bytesSent > 0 && bytesSent == packet.count {
        delegate?.simplePing(self, didSendPacket: packet, sequenceNumber: nextSequenceNumber)
    } else {
        let error = NSError(domain: NSPOSIXErrorDomain, code: Int(err != 0 ? err : ENOBUFS), userInfo: nil)
        delegate?.simplePing(self, didFailToSendPacket: packet, sequenceNumber: nextSequenceNumber, error: error)
    }
    
    nextSequenceNumber &+= 1
    if nextSequenceNumber == 0 {
        nextSequenceNumberHasWrapped = true
    }
}
```

### 接受消息
服务器返回消息后， Socket 调用 read 程序组件，将接受到响应消息存放在接受缓冲区中。
上面我们在创建套接字的时候，传入了一个 socketReadCallback 方法，并且限定了当返回类型是 CFSocketCallBackType.readCallBack 方法时回调我们的 callback 方法

Callback 方法中的逻辑很简单，进行一些判断后，调用 readData 方法：
```
private func socketReadCallback(s: CFSocket?,
                                type: CFSocketCallBackType,
                                address: CFData?,
                                data: UnsafeRawPointer?,
                                info: UnsafeMutableRawPointer?) {
    /* This C routine is called by CFSocket when there's data waiting on our ICMP
       * socket. It just redirects the call to Swift code. */
    // 转为 swift object
    let obj = unsafeBitCast(info, to: SimplePing.self)
    assert(obj.sock === s)
    
    assert(type == CFSocketCallBackType.readCallBack)
    assert(address == nil)
    assert(data == nil)
    
    obj.readData()
}
```
Readata 中的逻辑就是读取 IP 包，然后摘出其中的 ICMP，剖析 ICMP 返回包，检验其正确性之后将结果返回（下一段具体讲解析 ICMP 返回包的逻辑），如果成功，将 response 的内容和序列回调给代理，如果失败，抛错误给代理；
至此，就是一次完整的 Ping Pong
```
fileprivate func readData() {
    // 65535 is the maximum IP packet size
    let bufferSize = 65535
    // alloc 一片内存空间
    let buffer = UnsafeMutableRawPointer.allocate(byteCount: bufferSize, alignment: 0 /* We don’t need a specific alignment AFAICT */)
    defer {
        buffer.deallocate()
    }
    
    // 通过 recvfrom 方法读取数据
    let err: Int32
    // sockaddr_storage 既可以表示 IPv4 地址，也可以表示 IPv6 地址，是足以存储 IPv6 地址的结构体
    var addr = sockaddr_storage()
    var addrLen = socklen_t(MemoryLayout<sockaddr_storage>.size)
    let bytesRead = withUnsafeMutablePointer(to: &addr, { (addrStoragePtr: UnsafeMutablePointer<sockaddr_storage>) -> Int in
        let addrPtr = UnsafeMutablePointer<sockaddr>(OpaquePointer(addrStoragePtr))
        // 从指定套接字 sock 的 addrPtr 指针开始，读取长度为 addrLen 的数据到指定的 buffer 中
        return recvfrom(CFSocketGetNative(sock), buffer, bufferSize, 0, addrPtr, &addrLen)
    })
    // 失败则返回-1，错误原因存于errno中
    if bytesRead >= 0 {
        err = 0
    } else {
        err = errno
    }
    
    if bytesRead > 0 {
        var sequenceNumber = UInt16(0)
        var packet = Data(bytes: buffer, count: bytesRead)
        // 将 buffer 转给指向 IPv4Header 类型的指针，.pointee 实际上在取 buffer 中具体的值
        let ttl = buffer.assumingMemoryBound(to: IPv4Header.self).pointee.timeToLive
        // 将 packet 和 sequenceNumber 的引用传入，如果包是合法的，返回时这两个参数就已经被赋值
        if validatePingResponsePacket(&packet, sequenceNumber: &sequenceNumber) {
            delegate?.simplePing(self, didReceivePingResponsePacket: packet, sequenceNumber: sequenceNumber, ttl: ttl)
        } else {
            delegate?.simplePing(self, didReceiveUnexpectedPacket: packet)
        }
    } else {
        didFail(error: NSError(domain: NSPOSIXErrorDomain, code: Int(err != 0 ? err : EPIPE), userInfo: nil))
    }
}
```

### 解析 ICMP 返回包
分情况验证包的正确性，注意以下几次方法的参数都是  inout 的
```
func validatePingResponsePacket(_ packet: inout Data, sequenceNumber: inout UInt16) -> Bool {
    switch hostAddressFamily {
    case sa_family_t(AF_INET):
        return validatePing4ResponsePacket(&packet, sequenceNumber: &sequenceNumber)
    case sa_family_t(AF_INET6):
        return validatePing6ResponsePacket(&packet, sequenceNumber: &sequenceNumber)
    default:
        fatalError()
    }
}
```
下面只贴 IPv4 的验证逻辑，思路是一样的，只是 type 类型不一样
```
private func validatePing4ResponsePacket(_ packet: inout Data, sequenceNumber: inout UInt16) -> Bool {
    guard let icmpHeaderOffset = SimplePing.icmpHeaderOffset(in: packet) else {
        return false
    }
    
    let icmpPacket = Data(packet[icmpHeaderOffset...])
    // 传入 data 生成 ICMPHeader 结构体
    let icmpHeader = ICMPHeader(data: icmpPacket)
    
    let receivedChecksum = icmpHeader.checksum
    // 验证校验和
    let calculatedChecksum = UInt16(bigEndian: SimplePing.packetChecksum(packetData: icmpPacket))
    /* The checksum method returns a big-endian UInt16 */
    // 检验和不对，不合法
    guard receivedChecksum == calculatedChecksum else {
        return false
    }
    // 类型不是 0，代码不为 0，不合法，IPv4 的 ping 请求的 reply 对应的类型字段为 0， 代码字段为 0，下面实际贴了一张 ICMP 的 RFC 中的图
    guard icmpHeader.type == ICMPv4TypeEcho.reply.rawValue && icmpHeader.code == 0 else {
        return false
    }
    // 之前我们生成的 id，用于匹配包，不匹配则不合法
    guard icmpHeader.identifier == identifier else {
        return false
    }
    
    guard validateSequenceNumber(icmpHeader.sequenceNumber) else {
        return false
    }
    //
    packet = icmpPacket
    sequenceNumber = icmpHeader.sequenceNumber
    
    return true
}
```

IPv4 的 ping 请求的 reply 对应的类型字段为 0， 代码字段为 0：
![-w250](https://res.cloudinary.com/dp1pheuq7/image/upload/v1572099721/%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A7_2019-10-26_%E4%B8%8B%E5%8D%884.56.43_yt54kr.png)

```
// ICMP 包的偏移位置 
static private func icmpHeaderOffset(in ipv4Packet: Data) -> Int? {
    guard ipv4Packet.count >= IPv4Header.size + ICMPHeader.size else {
        return nil
    }
    
    // 通过 data 生成 IPv4 结构体
    let ipv4Header = IPv4Header(data: ipv4Packet)
    if ipv4Header.versionAndHeaderLength & 0xF0 == 0x40 /* IPv4 */ && Int32(ipv4Header.protocol) == IPPROTO_ICMP {
        // 取出第 1 个字节，与 0x0F 是为了取其中的后 4 位，因为首部长度字段只有 4 位
        let ipHeaderLength = Int(ipv4Header.versionAndHeaderLength & 0x0F) * MemoryLayout<UInt32>.size
        if ipv4Packet.count >= (ipHeaderLength + ICMPHeader.size) {
            return ipHeaderLength
        }
    }
    return nil
}
```
