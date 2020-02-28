##**稳定的 Java 堆 VS 动荡的 Java 堆**

一般来说，稳定的堆大小对垃圾回收是有利的。获得一个稳定的堆大小的方法是使-Xms 和-Xmx 的大小一致，即最大堆和最小堆 (初始堆) 一样。如果这样设置，系统在运行时堆大小理论上是恒定的，稳定的堆空间可以减少 GC 的次数。因此，很多服务端应用都会将最大堆和最小堆设置为相同的数值。但是，一个不稳定的堆并非毫无用处。稳定的堆大小虽然可以减少 GC 次数，但同时也增加了每次 GC 的时间。让堆大小在一个区间中震荡，在系统不需要使用大内存时，压缩堆空间，使 GC 应对一个较小的堆，可以加快单次 GC 的速度。基于这样的考虑，JVM 还提供了两个参数用于压缩和扩展堆空间。

-XX:MinHeapFreeRatio 参数用来设置堆空间最小空闲比例，默认值是 40。当堆空间的空闲内存小于这个数值时，JVM 便会扩展堆空间。

-XX:MaxHeapFreeRatio 参数用来设置堆空间最大空闲比例，默认值是 70。当堆空间的空闲内存大于这个数值时，便会压缩堆空间，得到一个较小的堆。

当-Xmx 和-Xms 相等时，-XX:MinHeapFreeRatio 和-XX:MaxHeapFreeRatio 两个参数无效。

##**增大吞吐量提升系统性能**

吞吐量优先的方案将会尽可能减少系统执行垃圾回收的总时间，故可以考虑关注系统吞吐量的并行回收收集器。在拥有高性能的计算机上，进行吞吐量优先优化，可以使用参数：

> –Xmx3800m –Xms3800m –Xmn2G –Xss128k –XX:+UseParallelGC –XX:ParallelGC-Threads=20 –XX:+UseParallelOldGC
   
–Xmx3800m –Xms3800m：设置 Java 堆的最大值和初始值。一般情况下，为了避免堆内存的频繁震荡，导致系统性能下降，我们的做法是设置最大堆等于最小堆。假设这里把最小堆减少为最大堆的一半，即 1900m，那么 JVM 会尽可能在 1900MB 堆空间中运行，如果这样，发生 GC 的可能性就会比较高；

-Xss128k：减少线程栈的大小，这样可以使剩余的系统内存支持更多的线程；

-Xmn2g：设置年轻代区域大小为 2GB；

–XX:+UseParallelGC：年轻代使用并行垃圾回收收集器。这是一个关注吞吐量的收集器，可以尽可能地减少 GC 时间。

–XX:ParallelGC-Threads：设置用于垃圾回收的线程数，通常情况下，可以设置和 CPU 数量相等。但在 CPU 数量比较多的情况下，设置相对较小的数值也是合理的；

–XX:+UseParallelOldGC：设置年老代使用并行回收收集器。

##**尝试使用大的内存分页**

CPU 是通过寻址来访问内存的。32 位 CPU 的寻址宽度是 0~0xFFFFFFFF ，计算后得到的大小是 4G，也就是说可支持的物理内存最大是 4G。但在实践过程中，碰到了这样的问题，程序需要使用 4G 内存，而可用物理内存小于 4G，导致程序不得不降低内存占用。为了解决此类问题，现代 CPU 引入了 MMU（Memory Management Unit 内存管理单元）。MMU 的核心思想是利用虚拟地址替代物理地址，即 CPU 寻址时使用虚址，由 MMU 负责将虚址映射为物理地址。MMU 的引入，解决了对物理内存的限制，对程序来说，就像自己在使用 4G 内存一样。内存分页 (Paging) 是在使用 MMU 的基础上，提出的一种内存管理机制。它将虚拟地址和物理地址按固定大小（4K）分割成页 (page) 和页帧 (page frame)，并保证页与页帧的大小相同。这种机制，从数据结构上，保证了访问内存的高效，并使 OS 能支持非连续性的内存分配。在程序内存不够用时，还可以将不常用的物理内存页转移到其他存储设备上，比如磁盘，这就是大家耳熟能详的虚拟内存。

在 Solaris 系统中，JVM 可以支持 Large Page Size 的使用。使用大的内存分页可以增强 CPU 的内存寻址能力，从而提升系统的性能。

> –Xmx2506m –Xms2506m –Xmn1536m –Xss128k –XX:++UseParallelGC –XX:ParallelGCThreads=20 –XX:+UseParallelOldGC –XX:+LargePageSizeInBytes=256m

–XX:+LargePageSizeInBytes：设置大页的大小。

过大的内存分页会导致 JVM 在计算 Heap 内部分区（perm, new, old）内存占用比例时，会出现超出正常值的划分，最坏情况下某个区会多占用一个页的大小。

##**使用非占有的垃圾回收器**

为降低应用软件的垃圾回收时的停顿，首先考虑的是使用关注系统停顿的 CMS 回收器，其次，为了减少 Full GC 次数，应尽可能将对象预留在年轻代，因为年轻代 Minor GC 的成本远远小于年老代的 Full GC。

> –Xmx3550m –Xms3550m –Xmn2g –Xss128k –XX:ParallelGCThreads=20 –XX:+UseConcMarkSweepGC –XX:+UseParNewGC –XX:+SurvivorRatio=8 –XX:TargetSurvivorRatio=90 –XX:MaxTenuringThreshold=31

–XX:ParallelGCThreads=20：设置 20 个线程进行垃圾回收；

–XX:+UseParNewGC：年轻代使用并行回收器；

–XX:+UseConcMarkSweepGC：年老代使用 CMS 收集器降低停顿；

–XX:+SurvivorRatio：设置 Eden 区和 Survivor 区的比例为 8:1。稍大的 Survivor 空间可以提高在年轻代回收生命周期较短的对象的可能性，如果 Survivor 不够大，一些短命的对象可能直接进入年老代，这对系统来说是不利的。

–XX:TargetSurvivorRatio=90：设置 Survivor 区的可使用率。这里设置为 90%，则允许 90%的 Survivor 空间被使用。默认值是 50%。故该设置提高了 Survivor 区的使用率。当存放的对象超过这个百分比，则对象会向年老代压缩。因此，这个选项更有助于将对象留在年轻代。

–XX:MaxTenuringThreshold：设置年轻对象晋升到年老代的年龄。默认值是 15 次，即对象经过 15 次 Minor GC 依然存活，则进入年老代。这里设置为 31，目的是让对象尽可能地保存在年轻代区域。