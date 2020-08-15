title: JVM最大线程数
date: 2013-06-17 22:32:29
categories: 技术
tags: [Java,线程]

------

>最近研发**推送方案**，需要大量线程来模拟手机客户端。在模拟手机客户端的时候，单个JVM线程数量始终卡在一万多一点，然后就报如下的错误`java.lang.OutOfMemoryError: unable to create new native thread`。我在网上找了很多资料，都是分析32位的，都是准备模拟几千个或者几万个水平。因为我是使用64位的linux,并且要模拟50万，发现很多都不是很详细，所以写了这篇文章。

对于这个OOM，根本无需多言，网上一堆关于JVM线程数的计算公式，此处我不妨再次引用一次:`(MaxProcessMemory - JVMMemory – ReservedOsMemory) / (ThreadStackSize) = Number of threads`。MaxProcessMemory：进程最大的寻址空间，关于不同系统的进程默认的最大寻址空间，可以参考下图。

![进程的寻址空间](/images/max-space-per-process.jpg)

<!--more-->


### JVM层面

* JVMMemory:Heap + PermGen
* ReservedOSMemory:Native heap，JNI

通过上面的说明大概就能估算出线程数量，公式如下:

`(MaxProcessMemory<固定值> – Xms<初始化值，最小值> – XX:PermSize<初始化值，最小值> – 100m<估算值>) / Xss = Number of threads<最大值> `

*-Xms，-Xmx，-Xss*，这三个变量就是JVM层面可以调节的。

----

### 系统层面:

#### /proc/sys/kernel/pid_max

**开始我也觉得奇怪，为什么需要修改pid呢？pid不是进程id嘛，我只是创建线程，并不创建进程，pid的大小跟我好像没有关系。查阅资料才发现，同一个进程的多个线程的pid其实是不同的。**

#### /proc/sys/vm/max_map_count

**max_map_count文件包含限制一个进程可以拥有的VMA(虚拟内存区域)的数量。虚拟内存区域是一个连续的虚拟地址空间区域。在进程的生命周期中，每当程序尝试在内存中映射文件，链接到共享内存段，或者分配堆空间的时候，这些区域将被创建。调优这个值将限制进程可拥有VMA的数量。限制一个进程拥有VMA的总数可能导致应用程序出错，因为当进程达到了VMA上线但又只能释放少量的内存给其他的内核进程使用时，操作系统会抛出内存不足的错误。如果你的操作系统在NORMAL区域仅占用少量的内存，那么调低这个值可以帮助释放内存给内核用。具体的可以查看redhat的这篇文章:<http://www.redhat.com/magazine/001nov04/features/vm/>，中文的话是这篇:<http://www.oschina.net/translate/understanding-virtual-memory?print>**

**注意：**ulimit -a中参数的大小

当然有人说下面这个两个需要修改，但是我测试下来，好像不用修改也行，反正我没有修改。

```
/proc/sys/kernel/thread-max
/etc/security/limits.d/90-nproc.conf

```

注意如果使用

* sysctl -w vm.max_map_count=200000
* sysctl -p

来修改，重启后值会改变，如果想永久改变需要采用如下形式。
* vm.max_map_count=200000直接写到/etc/sysctl.conf中
* 然后执行sysctl -p

对于我的应用，此处还要设置一个socket端口的取值范围

* net.ipv4.ip_local_port_range = 1024 65535

当然对于非socket应用是不需要设置该值的


>**最后说明**:上面各个参数的设置，可以去查看linux方面的一些资料。至于值到底多大，得根据你机器的性能来确定。
