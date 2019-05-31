---
layout:     post
title:      JVM 系列文章之 Full GC 和 Minor GC
subtitle:   Full GC 和 Minor GC 的区别
date:       2019-05-29
author:     mahjong
header-img: img/post-bg-swift.jpg
catalog: true
tags:
    - mahjong
---
#JVM 系列文章之 Full GC 和 Minor GC
## Full GC
**Full GC 就是收集整个堆，包括新生代，老年代，永久代(在JDK 1.8及以后，永久代会被移除，换为metaspace)等收集所有部分的模式**

针对HotSpot VM的实现，它里面的GC其实准确分类只有两大种：

- **Partial GC**：并不收集整个GC堆的模式
   - Young GC：只收集young gen的GC，**Young GC还有种说法就叫做 "Minor GC"**
   - Old GC：只收集old gen的GC。只有CMS的concurrent collection是这个模式
   - Mixed GC：收集整个young gen以及部分old gen的GC。只有G1有这个模式
- **Full GC**：收集整个堆，包括young gen、old gen、perm gen（如果存在的话）等所有部分的模式。

###Full GC的触发条件
针对不同的垃圾收集器，Full GC的触发条件可能不都一样。**按HotSpot VM的serial GC的实现来看**，触发条件是:

> - **当准备要触发一次 young GC时，如果发现统计数据说之前 young GC的平均晋升大小比目前的 old gen剩余的空间大，则不会触发young GC而是转为触发 full GC** (因为HotSpot VM的GC里，除了垃圾回收器 CMS的concurrent collection 之外，其他能收集old gen的GC都会同时收集整个GC堆，包括young gen，所以不需要事先准备一次单独的young GC)
> - **如果有永久代(perm gen),要在永久代分配空间但已经没有足够空间时，也要触发一次 full GC**
> - **System.gc()，heap dump带GC,其默认都是触发 full GC**。

HotSpot VM里其他非并发GC的触发条件复杂一些，不过大致原理与上面说的其实一样。

而**在 Parallel Scavenge 收集器下，默认是在要触发 full GC前先执行一次 young GC**,并且两次GC之间能让应用程序稍微运行一小下，以期降低 full GC的暂停时间 (因为 young GC 会尽量清理了young gen的死对象，减少了 full GC的工作量)。**控制这个行为的VM参数是: -XX:+ScavengeBeforeFullGC**。

并发GC的触发条件就不一样，**以 CMS GC为例，它主要是定时去检查old gen的使用量，但使用量超过了触发比例就会启动一次 CMS GC，对old gen做并发收集**。
##Minor GC
Minor GC 是俗称，**新生代(新生代分为一个 Eden区和两个Survivor区)的垃圾收集叫做 Minor GC**。
###触发条件
> 当 Eden 区的空间耗尽了怎么办？这个时候 Java虚拟机便会触发一次 Minor GC来收集新生代的垃圾，存活下来的对象，则会被送到 Survivor区。

**简单说就是当新生代的Eden区满的时候触发 Minor GC**

###Minor GC的过程
> 前面提到，新生代共有 两个 Survivor区，我们分别用 from 和 to来指代。其中 to 指向的Survivor区是空的。

> 当发生 Minor GC时，Eden 区和 from 指向的 Survivor 区中的存活对象会被复制(此处采用标记 - 复制算法)到 to 指向的 Survivor区中，然后**交换 from 和 to指针，以保证下一次 Minor GC时，to 指向的 Survivor区还是空的**。

**注意: from与to只是两个指针，它们变动的，to指针指向的Survivor区是空的**

### Survivor区对象晋升位老年代对象的条件
> Java虚拟机会记录 Survivor区中的对象一共被来回复制了几次。**如果一个对象被复制的次数为 15 (对应虚拟机参数 -XX:+MaxTenuringThreshold),那么该对象将被晋升为至老年代**，(至于为什么是 15次，原因是 HotSpot会在对象头的中的标记字段里记录年龄，分配到的空间只有4位，所以最多只能记录到15)。另外，**如果单个 Survivor 区已经被占用了 50% (对应虚拟机参数: -XX:TargetSurvivorRatio)，那么较高复制次数的对象也会被晋升至老年代**。

当Survivor区的部分对象晋升到老年代后，老年代的占用量通常会升高。

**注意：**

在Minor GC过程中，Survivor 可能不足以容纳Eden和另一个Survivor中的存活对象。如果Survivor中的存活对象溢出，多余的对象将被移到老年代，这称为**过早提升(Premature Promotion)**，这会导致老年代中短期存活对象的增长，可能会引发严重的性能问题。再进一步说，在Minor GC过程中，如果老年代满了而无法容纳更多的对象，Minor GC 之后通常就会进行Full GC,这将导致遍历整个Java堆，这称为**提升失败(Promotion Failure)**。至于解决办法，这就涉及到对应用程序的调优问题了，这里就不叙述了，如有兴趣，请自行查阅相关资料

### JVM如何避免Minor GC扫描全堆
HotSpot 给出的解决方案是 一项叫做 卡表 的技术。如下图所示:
![card table](http://ww1.sinaimg.cn/large/006tNc79ly1g3kjeumbfgj31lk0u0gru.jpg)
卡表的具体策略是将**老年代的空间分成大小为 512B的若干张卡，并且维护一个卡表，卡表本省是字节数组，数组中的每个元素对应着一张卡，其实就是一个标识位，这个标识位代表对应的卡是否可能存有指向新生代对象的引用**，如果可能存在，那么我们认为这张卡是脏的，即脏卡。如上图所示，卡表3被标记为脏。

**在进行Minor GC的时候，我们便可以不用扫描整个老年代，而是在卡表中寻找脏卡，并将脏卡中的老年代指向新生代的引用加入到 Minor GC的GC Roots里，当完成所有脏卡的扫描之后，Java 虚拟机便会将所有脏卡的标识位清零。这样虚拟机以空间换时间，避免了全表扫描**。

## 关于 Major GC的说明

除了Full GC和Minor GC外，还有一种说法叫做 "Major GC"：

> Major GC通常是跟full GC是等价的，收集整个GC堆，但因为 HotSpot VM发展这么多年，外界对各种名词的解读已经完全混乱了，当有人说"Major GC"的时候一定要问清楚他想要指的是上面的 full GC还是 old GC

以上是 R大关于 Major GC的说法，比较权威的。在网上还流行着**另外一种说法就是 Major GC是对老年代的垃圾回收**。


