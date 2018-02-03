# 深入理解java虚拟机 摘要 #
***
## 一、自动内存管理机制 ##
***
### 5.理解GC日志 ###


每一种收集器的日志形式都是由它们自身的实现所决定的，换而言之，每个收集器的日志格式都可以不一样。

```
33.125：[GC[DefNew：3324K-＞152K（3712K），0.0025925 secs]3324K-＞152K（11904K），0.0031680 secs]
1 0 0.6 6 7：[F u l l G C[T e n u r e d：0 K-＞2 1 0 K（1 0 2 4 0 K），0.0 1 4 9 1 4 2 s e c s]4603K-＞210K（19456K），[Perm：2999K-＞2999K（21248K）]，0.0150007 secs][Times：user=0.01 sys=0.00，real=0.02 secs]
```
<br>

* 最前面的数字“33.125：”和“100.667：”代表了GC发生的时间，这个数字的含义是从Java虚拟机启动以来经过的秒数。
<br>
* GC日志开头的“[GC”和“[Full  GC”说明了这次垃圾收集的停顿类型，而不是用来区分新生代GC还是老年代GC的。如果有“Full”，说明这次GC是发生了Stop-The-World的。如果是调用System.gc（）方法所触发的收集，那么在这里将显示“[Full GC（System）”。
<br>
* 接下来的“[DefNew”、“[Tenured”、“[Perm”表示GC发生的区域，这里显示的区域名称与使用的GC收集器是密切相关的
<br>
* 后面方括号内部的“3324K-＞152K（3712K）”含义是“GC前该内存区域已使用容量-＞GC后该内存区域已使用容量（该内存区域总容量）
<br>
* 再往后，“0.0025925 secs”表示该内存区域GC所占用的时间，单位是秒。有的收集器会给出更具体的时间数据，如“[Times：user=0.01 sys=0.00，real=0.02 secs]”，这里面的user、sys和real与Linux的time命令所输出的时间含义一致，分别代表用户态消耗的CPU时间、内核态消耗的CPU事件和操作从开始到结束所经过的墙钟时间（Wall Clock Time）。CPU时间与墙钟时间的区别是，墙钟时间包括各种非运算的等待耗时，例如等待磁盘I/O、等待线程阻塞，而CPU时间不包括这些耗时，但当系统有多CPU或者多核的话，多线程操作会叠加这些CPU时间，所以读者看到user或sys时间超过real时间是完全正常的。

#### 垃圾收集器参数 ####
|**参数**|**描述**|
|:---|:---|
|UseSerialGC|虚拟机运行在Client模式下的默认值，打卡此开关后，使用Serial + Serial Old的收集器组合进行内存回收|
|UseParNewGC|打开此开关后，使用ParNew + Serial Old 的收集齐组合进行内存回收|
|UseConcMarkSweepGC|打开次开关后，使用ParNew + CMS + Serial Old的收集器组合进行内存回收。Serial Old收集器讲作为CMS收集器出现 Concurrent Mode Failure 失败后的后备收集器使用|
|UseParalleGC|虚拟机运行在Server模式下的默认值，打开此开关后，使用Parallel Scavenge + Serial Old(Ps Marksweep)的收集器组合进行内存回收|
|SurvivorRatio|新生代种Eden区域与Survivor 区域的容量比值，默认为8，代表Eden"Survior = 8:1|
|PretenureSizeThreshold|直接晋升到老年代的对象大小，设置这个参数后，大于这个参数的对象将直接在老年代分配|
|MaxTenuringThreshold|晋升到老年代的对象年龄。每个对象在坚持过一次 Minor GC 之后，年龄就增加 1 ，当超过这个参数值时就进入老年代|
|UseAdaptiveSizePolicy|动态调整Java堆中各个区域的大小以及进入老年代的年龄|
|HandlePromotionFailure|是否允许分配担保失败，即老年代的剩余空间不足以应付新生代的整个Eden 和 Survivor 区的所有对象都存活的极端情况|
|ParallelGCThreads|设置并行GC时进行内存回收的线程数|
|GCTimeRatio|GC时间占总时间的比例，默认值为99，即允许1%的GC时间，仅在使用ParallelScavenge收集器时生效|
|MaxGCpauseMills|设置 GC 的最大停顿时间，仅在使用Parallel Scavenge收集器生效|
|CMSInitatingOccupancyFraction|设置CMS收集器在老年代空间被使用多少后触发垃圾手机。默认值为68%，仅在使用CMS收集器时生效|
|UseCMSCompactAtFullCollection|设置CMS收集器在完成垃圾收集后是否要进行一次内存碎片整理。仅在使用CMS收集器时生效|
|CMSFullGCsBeforeCompaction|设置CMS收集器在进行若干次垃圾收集后再启动一次内存碎片整理，仅在使用CMS收集器时生效|