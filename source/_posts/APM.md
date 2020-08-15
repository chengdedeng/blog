title: APM技术总结
date: 2016-12-04 14:00:00
categories: 技术
tags: [APM,Pinpoint,Dapper]
----

APM(Application Performance Management)经过2-3年的发展早已经被大家所熟悉了，当Google的Dapper刚发表的时候，就有一大批的公司和组织进行了产品化，大家熟悉的淘宝的鹰眼、点评的Cat、OneAPM等都是这方面的先行者。随着SOA和微服务的普及，越来越多的公司采用了这样的分布式架构，因此对APM的需求更是越来越强烈。本篇文章首先会简单阐述下Dapper的思想，然后会简单阐述下笔者自研的Dragon和放弃它的原因，最后阐述下Pinpoint架构。这篇文章主要的目的，一是记录研发的心路历程，二是希望为那些未做但想做的同行提供一点经验教训。

<!--more-->

### Dapper
**分布式系统** 演变到现在，用户的一次点击事件，导致后端几十次甚至上百次的同步或异步调用，这已经不再仅仅是Google、阿里、Facebook等级别的公司才会出现的情况。随着用户对产品的体验要求越来要高，任何的延迟或者抖动都会导致用户的流失，这对于任何的商业公司来说都是不能够被接收的现实。所以监控并且是7x24小时的不间断监控，是互联网企业的标配。Dapper的核心在于Call Tree模型，它抽象出了Trace、Span、Annotation，用简单的3个模型就将整个调用串起来了。Dapper的思想是非常简单明了的，但是Dapper却又提出了低损耗、应用透明、可扩展。个人觉得应用透明应该是非常难做的一件事情，不同的程序语言、不同的RPC框架、不同的线程调用模型，都是必须考虑的因素，当然采样率的控制、跟踪数据的收集存储、带外数据的携带、安全和隐私等等也都不是容易的事情。Dapper虽然代码没有开源，但是它的思想却为我们提供了非常高的起点，所以才有APM市场百花齐放的景象。

----

### Dragon
**[Dragon](https://github.com/chengdedeng/dragon)** 是我读完Dapper简单看了下Cat源码之后，针对使用Dubbo作为RPC的系统研发的一套调用链追踪系统。首先我不是一个非常喜欢造轮子的人，一是因为在当时没有针对Dubbo的调用链追踪工具，点评的Cat是基于点评内部的RPC框架Pigeon来研发的，二是Pinpoint我当时觉得是韩国人开发的，文档是一个问题，所以打算自己写了一套。当然这套东西最后被我放弃了，其实原因很简单因为我发现Pinpoint已经做的非常不错了，虽然文档还是比较烂，不过好在代码还比较靠谱，不懂的直接看代码，并且Dubbo的扩展也已经支持。之前凭借个人的判断，没有去仔细了解这个项目导致自己花了很多精力自研，如果当时就基于Pinpoint去实现Dubbo的扩展点就更完美了。关于自研和开源方案，笔者经常犯难，也做过很多错误的决定，但是任何事情都具有两面性，所以我能告诉你的就是大胆去做，及时改正。

虽然Dragon被我放弃了，但是不代表这是一个失败的项目或者说没有可取之处，正式由于Dragon的研发让我明白Pinpoint中的很多闪光之处和不足之处。Dragon在公司稳定运行了大概半年之久，发现和解决了不少的问题。Dragon的架构非常简单，实现了Dubbo Filter的扩展点，做了一个轻量级的AOP。Trace Id、Span Id都作为Dubbo Protocol的Attachment参数携带给RPC的下一跳。本跳的Trace数据被异步发送给Collector，Collector首先发送到消息队列，然后再入库。存储我选择了Mysql，当然是经过分库分表的，同时每晚有定时任务删除过期的数据保证数据量不至于太大。Web端根据时间和接口名去查询数据，最后绘制成调用链。当然异常数据会作为带外数据被收集起来，这个地方还可以作为插件的形式收集别的带外数据。最后如果用户发现一条Trace链路很慢，想看跨系统的所有的日志，只要将TraceId输入到我们kibana中就可以把所有的日志按时间序串起来。因为我修改了LOG4J的MDC，将TraceId放入到了线程的上下文中。为了形象的说明系统的样子，用图来讲比较方便说明。图中红色的都是表示有问题的，绿色表示正常。线上的时间表示网络耗时，左边的表示下行耗时，右边的表示上行耗时。每一个节点都可以点击，进去之后有该节点的地址和调用总耗时。

![dragon1](/images/dragon1.png)

![dragon2](/images/dragon2.png)

![dragon3](/images/dragon3.png)

![dragon4](/images/dragon4.png)

![dragon5](/images/dragon5.png)

有人可能看完这个之后，可能会问你这个只能查询微观粒度的，我怎么从宏观的角度去判断我的系统在这段时间不是很稳定或者慢了。其实我们还有一套Metrics系统，它是从宏观层面来反应我们所有系统的状态，如果它反应有问题并报警，再结合Dragon来定位就能够解决问题。

---

### Pinpoint

[Pinpoint](https://github.com/naver/pinpoint)通过Java Agent技术通过Attachment的方式进行了代码的织入，其实这并不是多高深的技术，可以详见我的文章《[JavaAgent的一点思路](http://www.yangguo.info/2015/07/18/JavaAgent%E7%9A%84%E4%B8%80%E7%82%B9%E6%80%9D%E8%B7%AF/)》。Pinpoint的架构和设计思路跟Dragon很像，不过它采用了Nosql的Hbase作为存储，数据使用TCP/UDP进行传送，并且可以通过Web UI实时获取Agent中的数据。它的架构见下图，不过稍微有点老了，暂且看看吧：

![pinpoint1](/images/pinpoint1.png)

对于监控数据或者非业务数据，我个人是非常喜欢使用UDP的，因为服务哪怕crash了，也不会对业务造成任何影响。虽然以上的功能已经让它看上去非常性感了，不过真正吸引我的却是它插件式的设计，可以根据自己的需求来研发任意插件。Pinpoint虽然有这样那样的优点，不过每个使用者都会觉得自己的一些需求还是没有满足，例如通过TraceId将所有系统的日志串起来，这就需要我们去修改部分源码。下面几张图，能够形象的展示Pinpoint的一些特色。

![pinpoint2](/images/pinpoint2.png)

![pinpoint3](/images/pinpoint3.png)

![pinpoint4](/images/pinpoint4.png)
