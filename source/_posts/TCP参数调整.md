title: TCP参数调整
date: 2013-10-30 10:22:30
categories: 技术
tags: [Linux,TCP]
------

>最近一直在进行手机推送服务器的研发，代码采用Java实现。netty作为一个优秀基于NIO的客户、服务器端编程框架，当然是首选的。server的研发必然是充满血与泪的，除了要熟悉Java的NIO，还得对tcp的封包、拆包，协议的设计、对象的序列化；压缩方式的选择等等之外，对Linux内核的调优是一件绝对比前面任何一个都要复杂。由于采用Java开发，由于Java的内存墙等诸多因素的影响，服务器支持的长连接我规划在一个server大概50W左右。可能对于一些C、C++服务器编程的码农来说这个链接数还真算不了什么，但是对于Java来说还是相当不错了。关于Java的内存墙方面的东西此处有一篇好文章推荐给大家，链接如下：<http://ifeve.com/jvm-performance-optimization-java-scalability-5>。既然一台服务器要支持50W的长连接 ，如果不对Linux内核进行调优，server的支撑的链接数和半链接的回收都会存在较大的问题。

前面YY了很多了，OK那我们下面进入正题。调优，顾名思义那就是碰见问题才调优，下面就是我碰见的**问题**和**解决方案**：

1.要支持50W的长连接，TCP栈对内存的使用需要调大。tcp_rmem和tcp_wmem我们都采用缺省值8KB，那么一个tcp链接，需要占用的内存大概为16KB。
一个简单的计算如下：接近8.0GB TCP内存能容纳的连接数，约为  8000000KB/16KB  = 50万。所以下面我分别配置成3G，8G，12G，一定注意单位是page，大小一般为4KB。

``` bash
>net.ipv4.tcp_mem = 786432 2097152 3145728
```
<!--more-->

2.由于手机网络不是很稳定，会经常出现网络闪断的情况，server端如何快速回收链接。tcp是可靠的传输协议，链接的关闭也需要4次握手，可能很多人对链接建立的3次握手非常熟悉 ，但是对正常关闭的4次握手却不熟悉。下面我附带一张图:
![tcp-close](/images/tcp-close.jpg)

此处我们思考的时候，把server和client的对调。当server端发现client端很久没有心跳，那我就得将该链接回收。由于Client端已经不可达，那server端链接会处在FIN-WAIT-1。这个时候该tcp链接已经是一个孤儿链接，也就是说它已经不属于任何一个进程。在不可达的情况下，它会默认发送9次，重试8次。由于该状态是非常占用资源的最大可占用64KB。所以我们得尽快让这个链接从FIN-WAIT-1中解放出来，所以设置如下：

``` bash
>net.ipv4.tcp_orphan_retries=1
```

3.由于手机网络不是很稳定，可能会出现数据包的乱跳；最主要的是NAT环境如LVS，很多公司都用LVS做负载均衡，通常是前面一台LVS，后面多台后端服务器，以NAT方式构建，当请求到达LVS后，它修改地址数据后便转发给后端服务器，但不会修改时间戳数据，对于后端服务器来说，请求的源地址就是LVS的地址，加上端口会复用，所以从后端服务器的角度看，原本不同客户端的请求经过LVS的转发，就可能会被认为是同一个连接，加之不同客户端的时间可能不一致，所以就会出现时间戳错乱的现象，于是后面的数据包就被丢弃了，具体的表现通常是是客户端明明发送的SYN，但服务端就是不响应ACK。就会出现一部分手机就是链接不上。

```
#是否启用时间戳选项，该选项会影响net.ipv4.tcp_tw_reuse，默认开启。一般我们为了安全我们不关闭该选项。
net.ipv4.tcp_timestamps = 1

#是否快速回收处于TIME_WAIT状态下得链接，默认关闭，最好别开启，除非有很懂的人建议。
net.ipv4.tcp_tw_recycle = 0

4.前面我们说了FIN-WAIT-1，那如何快速释放FIN-WAIT-2呢，虽然该状态没有FIN-WAIT-1那么耗资源。
net.ipv4.tcp_fin_timeout=30


其余的都不是很复杂了，最复杂的就是上面几个。我贴一份全的给大家参考：

# 开启TCP syncookies，防止DDOS攻击
net.ipv4.tcp_syncookies = 1
#syn报文（每个报文都需要排队）队列长度，超过该长度，请求就被丢弃，内存大于128M的默认为1024
net.ipv4.tcp_max_syn_backlog = 65536
#每个网络接口接收数据包的速率比内核处理这些包的速率快时，允许送到队列的数据包的最大数目
net.core.netdev_max_backlog = 32768
#定义了系统中每一个端口最大的监听队列的长度,这是个全局的参数
net.core.somaxconn = 32768  
#是否启用时间戳选项，该选项会影响net.ipv4.tcp_tw_reuse，默认开启
net.ipv4.tcp_timestamps = 1
#是否快速回收处于TIME_WAIT状态下的socket，由于手机网络时间戳会出现乱跳，所以必须关闭，这个默认关闭。
net.ipv4.tcp_tw_recycle = 0
#被动接受tcp链接时，第二次握手发送SYNACKs的次数，默认为5，对应的时间大概为180秒，官方说法。
net.ipv4.tcp_synack_retries = 3
#跟上面刚好相反，是主动发起tcp链接，发送SYNs的次数，默认为5，对应的时间大概为180秒，官方时间。
net.ipv4.tcp_syn_retries = 3
#我们关闭了TIME_WAIT快速回收，我们通过tcp_tw_reuse和tcp_max_tw_buckets来控制TIME_WAIT避免吃光机器，该值默认180000.
#如果服务器是作为客户端存在的，因为客户端连接受本地端口数限制，所以最好通过tcp_max_tw_buckets控制一下；如果服务器是作为服务端存在的，那么没有端口数的限制，只要情况允许，最好把tcp_max_tw_buckets设置大一些。纯粹就是防御dos攻击的，最好别认为降低该值。
net.ipv4.tcp_max_tw_buckets=180000
#开启处于TIME_WAIT态的socket重用，默认关闭。这个重用的是TIME_WAIT的端口，不是内存等，这个对客户端有意义。
net.ipv4.tcp_tw_reuse=1
#确定TCP栈如何使用内存，当大于上限是报文将丢弃。一般按照缺省值分配，上面的例子就是读写均为8KB，共16KB
#1.6GB TCP内存能容纳的连接数，约为  1600MB/16KB = 100K = 10万
#4.0GB TCP内存能容纳的连接数，约为  4000MB/16KB = 250K = 25万
net.ipv4.tcp_mem = 786432 2097152 3145728
#表示如果套接字由本端要求关闭，这个参数决定了它保持在FIN-WAIT-2状态的时间，2.2中默认180秒，之后默认为60秒  
net.ipv4.tcp_fin_timeout=30
#丢弃已经建立的tcp链接之前，需要多少次重试，默认15次，根据RTO的值，大概13-30分钟
net.ipv4.tcp_retries2=5
#放弃回应一个tcp连接请求之前，需要多少次重试，默认为3
net.ipv4.tcp_retries1=3
#收包速度大于内核处理包的速度时，输入队列最大报文数
net.core.netdev_max_backlog =  32768
#listen系统调用，最大的accept队列长度，超过该值时，后续请求被丢弃
net.core.somaxconn=32768
#针对孤立的socket（已经从进程上下文中删除，可是还有些清理工作没有完成），我们重试的最大次数。也就是server端close之后发[F.]的次数-1（0会重试一次），重负载服务器建议调小，默认为7。
net.ipv4.tcp_orphan_retries=1
```

-------
**PS：由于个人理解有限，如果不正确的请大家指正，以免误导他人，如果不明白原理，内核参数最好别动。**

-------
参考文章

<http://www.linuxinsight.com/>

<http://huoding.com/2012/01/19/142>
