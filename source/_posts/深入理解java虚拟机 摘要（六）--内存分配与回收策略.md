---
layout: post
title: "深入理解java虚拟机 摘要（六）--内存分配与回收策略"
subtitle: "自动内存管理机制"
date: 
author: "王鹏"
header-img: 
cdn: 'header-off'
tags: 
        - 'java虚拟机' 
        - 'java'
---
# 深入理解java虚拟机 摘要 #
***
## 一、自动内存管理机制 ##
***
### 6. 内存分配与回收策略 ###

&emsp;对象的内存分配，往大方向讲，就是在堆上分配（但也可能经过JIT编译后被拆散为标量类型并间接地栈上分配 ），对象主要分配在新生代的Eden区上，如果启动了本地线程分配缓冲，将按线程优先在TLAB上分配。少数情况下也可能会直接分配在老年代中，分配的规则并不是百分之百固定的，其细节取决于当前使用的是哪一种垃圾收集器组合，还有虚拟机中与内存相关的参数的设置。接下来我们将会讲解几条最普遍的内存分配规则，并通过代码去验证这些规则。

#### 对象优先在Eden分配 ####
大多数情况下，对象在新生代Eden区中分配。当Eden区没有足够空间进行分配时，虚拟机将发起一次Minor GC。
**代码示例:**
```
  private static final Integer _1MB = 1024*1024;
    /**
     *VM参数：-verbose：gc-Xms20M-Xmx20M-Xmn10M-XX：+PrintGCDetails
     -XX：SurvivorRatio=8
     */
    public static void testAllocation() {
        byte[] allocation1, allocation2, allocation3, allocation4;
        allocation1 = new byte[2 * _1MB];
        allocation2 = new byte[2 * _1MB];
        allocation3 = new byte[2 * _1MB];
        allocation4 = new byte[4 * _1MB];//出现一次Minor GC
    }
    public static void main(String[] args){
        testAllocation();
    }
```
**启动参数:**
```
-XX:+PrintGCDetails -Xms20M -Xmx20M -Xmn10M -XX:SurvivorRatio=8
```
&emsp;-Xms20M、-Xmx20M、-Xmn10M这3个参数限制了Java堆大小为20MB，不可扩展，其中10MB分配给新生代，剩下的10MB分配给老年代。
&emsp;-XX：SurvivorRatio=8决定了新生代中Eden区与一个Survivor区的空间比例是8:1
**运行结果：**
```
[GC (Allocation Failure) [PSYoungGen: 7414K->1016K(9216K)] 7414K->5188K(19456K), 0.0095543 secs] [Times: user=0.00 sys=0.00, real=0.01 secs] 
Heap
 PSYoungGen      total 9216K, used 7398K [0x00000000ff600000, 0x0000000100000000, 0x0000000100000000)
  eden space 8192K, 77% used [0x00000000ff600000,0x00000000ffc3bb60,0x00000000ffe00000)
  from space 1024K, 99% used [0x00000000ffe00000,0x00000000ffefe010,0x00000000fff00000)
  to   space 1024K, 0% used [0x00000000fff00000,0x00000000fff00000,0x0000000100000000)
 ParOldGen       total 10240K, used 4172K [0x00000000fec00000, 0x00000000ff600000, 0x00000000ff600000)
  object space 10240K, 40% used [0x00000000fec00000,0x00000000ff013160,0x00000000ff600000)
 Metaspace       used 3484K, capacity 4496K, committed 4864K, reserved 1056768K
  class space    used 381K, capacity 388K, committed 512K, reserved 1048576K
```
&emsp;从输出的结果也可以清晰地看到“eden space  8192K、from  space  1024K、to  space  1024K”的信息，新生代总可用空间为9216KB（Eden区+1个Survivor区的总容量）。
&emsp;执行testAllocation（）中分配allocation4对象的语句时会发生一次Minor  GC，这次GC的结果是新生代7414K变为1016K，而总内存占用量则几乎没有减少（因为allocation1、allocation2、allocation3三个对象都是存活的，虚拟机几乎没有找到可回收的对象）。这次GC发生的原因是给allocation4分配内存的时候，发现Eden已经被占用了6MB，剩余空间已不足以分配allocation4所需的4MB内存，因此发生Minor  GC。GC期间虚拟机又发现已有的3个2MB大小的对象全部无法放入Survivor空间（Survivor空间只有1MB大小），所以只好通过分配担保机制提前转移到老年代去。
<br>
#### (1)大对象直接进入老年代 ####
<br>
&emsp;所谓的大对象是指，需要大量连续内存空间的Java对象，最典型的大对象就是那种很长的字符串以及数组（笔者列出的例子中的byte[]数组就是典型的大对象）。大对象对虚拟机的内存分配来说就是一个坏消息（替Java虚拟机抱怨一句，比遇到一个大对象更加坏的消息就是遇到一群“朝生夕灭”的“短命大对象”，写程序的时候应当避免），经常出现大对象容易导致内存还有不少空间时就提前触发垃圾收集以获取足够的连续空间来“安置”它们。
&emsp;虚拟机提供了一个-XX：PretenureSizeThreshold参数，令大于这个设置值的对象直接在老年代分配。这样做的目的是避免在Eden区及两个Survivor区之间发生大量的内存复制（复习一下：新生代采用复制算法收集内存）。
&emsp;注意PretenureSizeThreshold参数只对Serial和ParNew两款收集器有效，Parallel  Scavenge收集器不认识这个参数，Parallel  Scavenge收集器一般并不需要设置。如果遇到必须使用此参数的场合，可以考虑ParNew加CMS的收集器组合。
<br>
#### (2)长期存活的对象将进入老年代 ####
<br>
&emsp;既然虚拟机采用了分代收集的思想来管理内存，那么内存回收时就必须能识别哪些对象应放在新生代，哪些对象应放在老年代中。为了做到这点，虚拟机给每个对象定义了一个对象年龄（Age）计数器。如果对象在Eden出生并经过第一次Minor GC后仍然存活，并且能被Survivor容纳的话，将被移动到Survivor空间中，并且对象年龄设为1。对象在Survivor区中每“熬过”一次Minor GC，年龄就增加1岁，当它的年龄增加到一定程度（默认为15岁），就将会被晋升到老年代中。对象晋升老年代的年龄阈值，可以通过参数-XX：MaxTenuringThreshold设置。
#### (3)动态对象年龄判定 ####
&emsp;为了能更好地适应不同程序的内存状况，虚拟机并不是永远地要求对象的年龄必须达到了MaxTenuringThreshold才能晋升老年代，如果在Survivor空间中相同年龄所有对象大小的总和大于Survivor空间的一半，年龄大于或等于该年龄的对象就可以直接进入老年代，无须到MaxTenuringThreshold中要求的年龄。
#### (4)空间分配担保 ####
&emsp;在发生Minor GC之前，虚拟机会先检查老年代最大可用的连续空间是否大于新生代所有对象总空间，如果这个条件成立，那么Minor GC可以确保是安全的。如果不成立，则虚拟机会查看HandlePromotionFailure设置值是否允许担保失败。如果允许，那么会继续检查老年代最大可用的连续空间是否大于历次晋升到老年代对象的平均大小，如果大于，将尝试着进行一次Minor GC，尽管这次Minor GC是有风险的；如果小于，或者HandlePromotionFailure设置不允许冒险，那这时也要改为进行一次Full GC。
&emsp;下面解释一下“冒险”是冒了什么风险，前面提到过，新生代使用复制收集算法，但为了内存利用率，只使用其中一个Survivor空间来作为轮换备份，因此当出现大量对象在Minor GC后仍然存活的情况（最极端的情况就是内存回收后新生代中所有对象都存活），就需要老年代进行分配担保，把Survivor无法容纳的对象直接进入老年代。与生活中的贷款担保类似，老年代要进行这样的担保，前提是老年代本身还有容纳这些对象的剩余空间，一共有多少对象会活下来在实际完成内存回收之前是无法明确知道的，所以只好取之前每一次回收晋升到老年代对象容量的平均大小值作为经验值，与老年代的剩余空间进行比较，决定是否进行Full GC来让老年代腾出更多空间。