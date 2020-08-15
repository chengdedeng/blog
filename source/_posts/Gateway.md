title: API Gateway的开源解决方案那么多，为什么我们却还要选择自研？
date: 2018-01-11 09:00:00
categories: 技术
tags: [Http,TLS,Socks5,Proxy,ShadowSocks,Netty,LittleProxy]

-----

>API Gateway/Backend for Front-End作为一种目前非常流行并且经过验证的Pattern，不论是在Netflix/Amazon还是BAT都得到了广泛的应用。在[Microservice architecture pattern](https://en.wikipedia.org/wiki/Microservices)大行其道的当下，API Gateway的建设显得尤为重要，本文主要是分享笔者在API Gateway研发中的一些心得体会，同时这篇文章也发布在了公众号**聊聊架构**，可以移步[此处](https://mp.weixin.qq.com/s/OSzGUse2yBFHgmGhSVhBIw)查看讨论。

--------

### Context
为了阐述API Gateway建设带来的收益是值得我们投入大成本来研发的，我们假设需要建设一个**online store**。为了支撑整个电子商务逻辑的运转，可以简单抽象成如下的微服务：

1. shopping cart
2. shipping
3. inventory
4. recommendation
5. order
6. review
7. product catalog

架构如下图所示：

![gateway architecture](/images/gateway1.png)

<!--more-->

Gateway的职责如下：
1. 安全
2. 流控
3. 缓存
4. 鉴权
5. 监控
6. 日志
7. 协议转换
8. .....

可以想象在**Microservice**的架构体系之下，如果没有统一的地方去处理这些跟业务关联度不是很紧密但是又必不可少的逻辑，那么每个上面需求将会在每个**Microservice**都实现一遍，这种代价非常的高昂。当然任何事物都存在双面性，有好就必然有坏的一面，流量全部都要经过Gateway，所以它一定要非常稳定且高可用，假如一旦此处出现问题，那么可以想象问题的严重性。

--------

### 技术选型
技术选型从来就不是一件容易的事情，虽然笔者没有所谓的选择恐惧症，但是想到要拦截所有的7层流量，心里面还是难免有点犯怵。业界开源的方案有[Netflix/zuul](https://github.com/Netflix/zuul)，[Kong/kong](https://github.com/Kong/kong)，[openresty/openresty](https://github.com/openresty/openresty)+`自研`，[TykTechnologies/tyk](https://github.com/TykTechnologies/tyk)，[Netty](https://github.com/netty/netty)+`自研`。这些都是非常不错的方案，并且已经经过了大量的生产实践，那么剩下的事情就只有根据自身的实际情况来选择。经过仔细分析，我们希望API Gateway足够薄，不希望在其上附加太多的功能。如果单纯的从性能的角度来考虑，**Kong** / **Openresty** 估计是最好的，毕竟底层都是Nginx。虽然在Openresty平台上做过一些小的模块，但是毕竟在这个上面算不上专家。**Zuul** 是Java平台上非常好的选择之一，有Netflix和Pivotal的支持并且功能强大，但是考虑到跟Netflix的全家桶集成很紧密，组件很多且复杂度较高，我们的系统主要还是架构的`Dubbo`之上，Spring Cloud用的地方不多。该项目的发起经历了两个阶段，初期我们是想做[WAF](https://en.wikipedia.org/wiki/Web_application_firewall)把我们系统的安全给提升上来，并没有一上来就想做API Gateway，虽然这两者从整体架构上来说是没有本质上的差异。随着WAF的研发完成，我们觉得这一层可以做更多的事情，这才有了API Gateway的构想。随后我们在WAF的基础之上逐步新增了很多功能，就如后面我们会讨论的架构图所示，WAF的拦截最终变成了我们Filter中的一环。当时由于笔者有多年使用`Netty`的经验，所以我们在设计WAF是就选用Netty。

考虑到**Netty**还是太偏底层，所以选择了构建在**Netty**之上的[LittleProxy](https://github.com/adamfisk/LittleProxy)作为我们的HTTP Proxy，这样就避免了我们完全重头开发。**LitteProxy** 的Programmers也是[LANTERN](https://www.getlantern.org/)的维护者。关于蓝灯的介绍，可以移步[维基百科](https://zh.wikipedia.org/wiki/%E8%93%9D%E7%81%AF)。LittleProxy只有二十来个类十来个接口，从头到尾通读一遍代码也就2/3天的时间，并且从GitHub Star数和Stack Overflow的收录数量来看，它都是不错的选择。作为API Gateway中最重要的部分确定之后，其实剩下的困难点就不多了。

如果当时我们一开始就想好了要构建一套完整的API Gateway，是不是现在就是例外的选择了，也许是也许不是，但是这些都不重要，重要的是选择了就得尽最大的努力让你别为自己的选择而后悔！

### 架构设计
在讨论我们的架构设计之前，先让我们来欣赏下zuul的架构，如下图所示：

![zuul architecture](/images/gateway2.png)

上图中`Zuul Servlet`是Http流量的接入点，Zuul从2.x和3x分支也开始使用Netty作为它们网络通讯的核心。`ZuulFilter Runner`是Gateway的业务核心，Filter type分为三种类型，`Pre routing`/`Routing`/`Post routing`，其实就是我们所谓的[AOP](https://en.wikipedia.org/wiki/Aspect-oriented_programming)。虽然ZuulFilter被分为了三种类型，但是其实它们是共享一个Request Content，这一点非常重要，ZullFilter的执行流程如下图：

![ZuulFilter](/images/gateway3.png)

一个HTTP Request进来首先要通过pre filter的检查，如果没有任何问题，就交给routing filter让其把流量转发给origin server，然后origin server会返回响应结果给下游的post filter。以上两步任何一步出现问题就直接到error filter，error filter也会将流量转给post filter。最终都是由post filter来对response进行整形，然后返回给上游的代理，最终到达用户。Zuul为了方便用户定制并且动态化加载配置引入了Groovy，这是非常好的地方，即我们享受到静态化的性能，同时又能享受到动态化的灵活。右侧和底部的设计是用来管理Zuul相关配置的，包含服务的注册发现、Filter的管理、统计分析相关的很多东西。

分析完Zuul之后，我们丛中吸收了很多东西，下图是我们的架构：

![waf architecture](/images/gateway4.jpg)

从上图可以看出，我们精简了Zuul从而更关注其核心的部分，因为我们的目的不是构建一套功能强大的中间件，而是一套非常精简的API Gateway。

-------------

### 研发过程
WAF我已经开源出来，有兴趣的朋友可以把玩下，地址为[https://github.com/chengdedeng/waf](https://github.com/chengdedeng/waf)。该项目非常精简，真正核心的类估计也就5个左右，非常适合阅读和练习。关于WAF的技术过程中遇到的问题，我也写了两篇文章来说明，详见[Java版WAF实现细节](https://www.yangguo.info/2017/06/06/Java版WAF实现/)和[HttpProxy研发心得](https://www.yangguo.info/2017/11/13/HttpProxy研发心得/)。WAF中的有些特性其实在API Gateway中用到的概率较小，比如ShadowSocks/Socks5的支持，如果这些大家不需要可以移除，这样代码就更少更精简。如果想让WAF要真正的成为API Gateway，还有很多的事情要做，例如Zuul中groovy自定义Filter以及管理功能，这些都需要根据需求定制。我们内部的版本也正在逐步完善，一些我觉得有用的特性也会同步到WAF中去。目前我们的API Gateway还非常简单，例如路由我们目前只支持Host，不支持URL的regex，而且很多管理功能都还处于开发阶段，但是从目前运行的情况来看还是非常稳定。

对于自研Gateway，其实难度还是不小，并不像看上去的那么美。如果对网络编程不是很熟悉，特别对TCP/HTTP理解不是比较全面，建议要慎重，因为这样风险太大。我们在研发过程就碰见很多的问题，下面就简单的列举几个：
1. HTTP Header头的大小写问题
2. CORS OPTIONS不受XHR的控制
3. Netty堆外内存泄漏
4. Origin Server心跳检测
5. Request Content-Type数据传递的区别
6. 根据URL Regex路由
7. ......

### 总结
网络上关于kong/zuul的文章非常多，关于自研的却不是很多，所以分享出来供大家参考，希望大家在做方案的时候有更多的选择和思考。
