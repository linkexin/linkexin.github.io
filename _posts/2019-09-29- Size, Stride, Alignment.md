---
title: Size, Stride, Alignment
---

当我们从内存层面去处理 Swift 属性时，我们就绕不过这三个属性：Size、Stride 和 Alignment

## Size
来看两个简单的 struct
```
struct Year {
  let year: Int
}

struct YearWithMonth {
  let year: Int
  let month: Int
}
```
直觉告诉我 YearWithMonth 的实例所占的内存更大，为了更严谨，我们需要验证这一点，but how？

### MemoryLayout
MemoryLayout 是 Swift3.0 推出的一个工具类，用来计算数据占用内存的大小
可以直接通过数据类型得到大小：
```
let size = MemoryLayout<Year>.size
```
也可通过实例变量得到实例所属类型的大小：
```
let instance = Year(year: 1984)
let size = MemoryLayout.size(ofValue: instance)
```
在上面两个例子中，Size 都是 8 bytes，YearWithMonth 的 Size 是 16 bytes

### 回到 Size
计算 struct size 的方式，凭直觉就是把每个属性的 size 相加得到一个总和，比如下面这个 struct：
```
struct Puppy {
  let age: Int
  let isTrained: Bool
}
```
按照上面的思路，它的 size 就应该是：
```
MemoryLayout<Int>.size + MemoryLayout<Bool>.size
// returns 9, from 8 + 1

MemoryLayout<Puppy>.size
// returns 9
```
在这个 Case 中确实是这样，看起来很完美，但是其实会有不一样的情况，下面会讲到

## Stride
当你在单个 buffer（例如数组）中处理多个实例时，类型的 Stride 尤其重要。
Stride 中文可以理解为「跨度」
如果我们有一个包含 puppy 结构体的数组，每个 puppy 都是 9 bytes，那么它们在内存是怎么存储的？也许是这样：
![-w250](https://res.cloudinary.com/dp1pheuq7/image/upload/v1569728771/stride-nopadding_xkv30s.png)
但是事实并非如此

Stride 决定了两个相邻的元素之间的距离，也就是元素从开始地址到结束地址所占用的连续内存字节的大小，这个数值会大于等于 Size，stride - size 个字节则为每个元素因为内存对齐而浪费的内存空间。
```
MemoryLayout<Puppy>.size
// returns 9

MemoryLayout<Puppy>.stride
// returns 16
```

所以布局实际上是这样的：
![-w250](https://res.cloudinary.com/dp1pheuq7/image/upload/v1569728771/stride-padding_m4wlpt.png)
如果你有一个指向第一个元素的字节指针，现在想移动到第二个元素，那么 Stride 的数值就是你的字节指针需要移动的字节距离

为什么 Size 和 Stride 会不一样，下面会讨论到

## Alignment
想象一下计算机一次获取 8bit 或者是 1byte 的内存数据，访问 bit1 和 bit7 的所需的时间是相同的 （不知道图里为什么标的是 bytes）
![-w250](https://res.cloudinary.com/dp1pheuq7/image/upload/v1569728772/alignment-byte8_syedzn.png)

现在你的电脑升级成了 16 位，每次可以读取 16bit 的数据，我们称之为 words，你仍然使用旧的软件，并且想以字节访问数据：如果你想读取 byte0 和 byte1 的数据，计算机只需要访问一次内存，把 16bit 的数据一次性读取出来，然后再拆分成前 8bit 和后 8bit，也就是我们需要的 byte0 和 byte1，在这种理想情况下，字节级内存访问速度是原来的两倍！
![-w250](https://res.cloudinary.com/dp1pheuq7/image/upload/v1569728771/alignment-byte16_gpvge6.png)

现在我们要说一种可能会发生在 16bit 机器上的异常现象
![-w250](https://res.cloudinary.com/dp1pheuq7/image/upload/v1569728770/alignment-misaligned16_zyc1so.png)

假如你在 16 位机上想访问 byte3 的数据，现在面临的问题就是你想访问的数据的排列是没有校准的，为了读取它，计算机需要访问位置 1 的 words，然后把它拆成两半，再读位置 2 的 words，再拆成两半，最后将前后两半拼起来，两次分离的 16bit 内存读取，最终只是为了获取一个 16bit 数据，这样效率比之前慢了两倍！
这可能还是好的情况，在一些操作系统上，这么做会直接导致崩溃。
这也就是为什么，我们需要对齐（校正）内存，Alignment 这个属性就是与内存对齐相关的属性。许多计算机系统对基本数据类型的合法地址做出了一些限制，要求某种数据类型对象的地址必须是某个值 **K** (通常是 2、4 或者 8 ) 的倍数。这种对齐限制简化了处理器和内存系统之间接口的硬件设计。**对齐原则是任何 K 字节的基本对象的地址必须是 K 的倍数。**

在 Swift 中，像 Int、Double 这种简单类型的 Alignment 和 Size 是一致的，32 位整型的 Size 是 4byte，并且需要校正到 4byte
```
MemoryLayout<Int32>.size
// returns 4
MemoryLayout<Int32>.alignment
// returns 4
MemoryLayout<Int32>.stride
// returns 4
```

### 复合类型
现在回到 puppy 结构体，它有一个 Int 和一个 Bool 属性，再次想想在 buffer 中这些值是怎么布局的：
![-w250](https://res.cloudinary.com/dp1pheuq7/image/upload/v1569728770/alignment-nopadding-bytes_edyxt5.png)
Bool 值特别开心，因为它的 Alignment = 1，所以它在哪个位置，地址都符合 Alignment 的倍数，但是第二个值 Int 类型就没有校正了，64 位机器的 Int 型的 Alignment = 8，但是可以看到图上的 Int 值并不符合起始内存地址是 8 的倍数。
请记住 puppy 类型的 stride = 16，这就意味着 buffer 实际上是：
![-w250](https://res.cloudinary.com/dp1pheuq7/image/upload/v1569728771/stride-padding_m4wlpt.png)
我们满足了 struct 内部对所有值的对齐要求：第二个整数位于字节 16 处，是 8 的倍数。
这就是 struct 的 Stride 可以大于 Size 的原因：需要添加足够的填充来满足对齐要求。

### 计算 Alignment
puppy 结构体的 Alignment 是多少？
```
MemoryLayout<Puppy>.alignment
// returns 8
```

Struct 的 Alignment 就是其所有属性中的最大的 Alignment。 在 Int 和 Bool 之间，Int 的 Alignment较大，为 8，因此 struct 的 Alignment = 8，仔细想想可以明白：基础类型的 Alignment 通常是 2、4 或者 8，取最大值也就是 8，但凡是  8 的倍数，肯定也是 2 和 4 的倍数，所以取所有属性中的最大值可以满足所有内部属性 Alignment 的合法要求
接着，Stride 将变为 Size 四舍五入到下一个 Alignment 倍数的大小。 在我们的情况下：
Size = 9，9 不是 8 的倍数，下一个 8 的倍数是 16，因此 Stride = 16

### 最后一个难题
我们有原始的 Puppy struct 和变化后的 AlternatePuppy struct：
```
struct Puppy {
  let age: Int
  let isTrained: Bool
} // Int, Bool

struct AlternatePuppy { 
  let isTrained: Bool
  let age: Int
} // Bool, Int
```

AlternatePuppy  的 Alignment = 8，Stride = 16，但是
```
MemoryLayout<AlternatePuppy>.size
// returns 16
```

Bool 后面跟着一个 Int，内存结构也许是这样的：
![-w250](https://res.cloudinary.com/dp1pheuq7/image/upload/v1569728770/alignment-internal-1_o55p5b.png)
实际上并不是，你应该发现了问题，8byte 整型没有对齐，它在 1 位置上，而 1 不是 alignment = 8 的倍数，正确的内存结构应该是这样的：
![-w250](https://res.cloudinary.com/dp1pheuq7/image/upload/v1569728771/alignment-internal-2_larvl1.png)
需要明确的是 struct 和内部的属性都要对齐，属性之间的内存填充会将整个 struct 的内存扩大，但是两个 struct 之间的内存填充不会扩大 struct 本身的 Size，这也就是为什么 Puppy 的 size 是 9，AlternatePuppy 的 size = 16，因为 Puppy 的填充是 struct 之间的，不算做 struct 的内存，AlternatePuppy 的填充是 struct 内部的属性之间的，会被算作是 struct 的内存
在这个例子中，stride 仍然是 16，Puppy 和 AlternatePuppy 实际上的变化来自于属性之间内存填充的存在

## 总结
最后，一个指针 UnsafeRawPointer (也就是 C 中的 void *)，你知道这个指针指向的元素类型，那么 Size、Stride 和 Alignment 分别是什么样的？
* Size 是读取到当前实例所有数据的字节数
* Stride 是从当前实例前进到下一个实例的字节数
* Alignment 是每个实例必须位于的「均匀可除数」数字，如果要为复制数据而分配内存，则需要指定正确的对齐方式（例如，allocate（byteCount：100，alignment：4））
![-w250](https://res.cloudinary.com/dp1pheuq7/image/upload/v1569728770/size-stride-alignment-summary_qluqvc.png)

对于我们大多数人而言，大多数时候，我们可能会处理诸如数组和集合之类的高级集合，而无需考虑底层的内存布局。在其他情况下，你可以在平台上使用较底层的 API 或者和 C 交互。 
如果你有一个 Swift 结构数组，并且需要 C 来读取它，则需要以正确的对齐方式分配一个 buffer，确保结构内部的填充对齐，并确保正确的 Stride，这样才能正确的访问数据
就像上面说到的，即使计算大小也不像看起来那么简单，每个属性的大小和对齐方式之间存在一些相互作用，这些因素决定了结构的整体大小。

## Mention
这篇文章大部分是对 [Size, Stride, Alignment - Swift Unboxed](https://swiftunboxed.com/internals/size-stride-alignment/) 的翻译，其中添加了一下自己的理解。
