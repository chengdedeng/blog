title: Java版WAF技术细节
date: 2017-06-06 16:03:30
categories: 技术
tags: [架构、HTTP、安全]
------

最近一直在研究应用安全相关的技术和产品，因为随着产品不断推广放量，安全体系的建设已经刻不容缓。谈到应用安全大家想到最多就是各种商业级的WAF防火墙，但是这个不是我想谈的重点，这次我们主要谈谈软件WAF。说到软件WAF，大家最熟悉的就是[ModSecurity](https://modsecurity.org/)、[esapi-java-legacy](https://github.com/ESAPI/esapi-java-legacy)、[lua-resty-waf](https://github.com/p0pr0ck5/lua-resty-waf)、[ngx_lua_waf](https://github.com/loveshell/ngx_lua_waf)等众多解决方案。虽然笔者所在公司使用了以上方案中的一种，但是为了研究和学习，笔者用Java重新造了一个轮子，本篇文章主要是阐述笔者设计和研发的思路，供大家参考。项目地址为：https://github.com/chengdedeng/waf。


<!--more-->

### HTTP代理
上面提到的软件WAF，除了esapi-java-legacy之外，其余几种都是基于代理模式。代理的优点就在于对WEB应用没有任何侵入，对业务的干扰几乎为零，这种方案是毫无疑问是最应该被采纳的。ModSecurity基于Apache，lua-resty-waf和ngx_lua_waf基于Nginx，Apache和Nginx作为业界最优秀的开源Proxy，它们的性能及稳定性是毫无疑问的。Java之上要想构建一套稳定可靠的网络通讯，必然绕不开Netty，所以在HTTP代理层面我选择了基于Netty研发的[LittleProxy](https://github.com/adamfisk/LittleProxy)，它的programmers也是[LANTERN](https://www.getlantern.org/)的维护者。关于蓝灯的介绍，可以移步[维基百科](https://zh.wikipedia.org/wiki/%E8%93%9D%E7%81%AF)。


HTTP代理分为正向和反向两种，作为应用防火墙，肯定要选择反向模式。由于LittleProxy原生支持Proxy Chain，所以WAF提供了透明模式和反向代理两种模式。透明模式就是使用LittleProxy的ChainProxyManager，把真正目标机的映射交给下游的Proxy，反向代理就是WAF直接管理目标机的映射关系。透明代理最为简单，只需要实现ChainedProxyManager就行，但是对于反向代理却需要考虑loadbalance和心跳检测。特别需要注意的是，当节点不健康的时候需要将节点从路由表中移除，当节点恢复健康状态，要能够及时的将节点新增到路由表,在WAF中的具体实现可以参考[HostResolverImpl](https://github.com/chengdedeng/waf/blob/master/src/main/java/info/yangguo/waf/HostResolverImpl.java)。

由于HTTP1.1支持chunked，对于client传递上来的数据别尝试配置较大的buffer，从而获取FullHttpRequest。因为当有大文件上传时，当你配置的buffer放不整个文件时，就会导致请求失败，在Nett的ChannelPipeline上也需要移除**inflater**和**aggregator**两个handler。为了支持TLS，可以自己实现SslEngineSource，WAF中的实现可以参考[SelfSignedSslEngineSource2](https://github.com/chengdedeng/waf/blob/master/src/main/java/info/yangguo/waf/util/SelfSignedSslEngineSource2.java)，WAF的证书是自签名的，大家可以换成CA签名的即可，浏览器就不会报不可信任了。

要写一个稳定可靠的HTTP代理服务器，虽然已经有了Netty和LittleProxy，但是还远远不够。一定要仔细研究HTTP协议，开源程序只是协议的实现，协议才是核心中的核心。因为一旦对协议理解的不是很清楚，你就可能给自己埋一个坑，所以在协议的研究上多花点时间是绝对值得，并且可以取得事半功倍的效果。

### 安全
应用程序的数据安全，在WAF中被分为两种类型，一种是进来的数据另外一种是出去的数据，也就是Request和Response，它们分别对应**HttpRequestFilterChain**和**HttpResponseFilterChain**责任链。

RequestFilter又分为黑名单Filter和白名单Filter，Request拦截又分为黑白名单两种,Response拦截主要给输出的数据进行安全加固.在Request的拦截规则方面,我参考了loveshell/ngx_lua_waf.
Request Content-Type又为`form-data`,`x-www-form-urlencoded`、`raw`、`binary`等多种类型，所以这里又牵涉到是否需要对url decode，同时还要考虑大小写的问题，只有方方面面了解透了，才能做到较高质量的防护。Response作为一种可扩展的增强，例如已经有不可信的数据进入系统，拦截已经晚了，这时我们就可以通过Response进行转义之后输出，或者做一些例如防止Clickjack的攻击。安全方面因为笔者能力和精力有限，只做了极少的一部分，需要大家仔细阅读例如[OWASP_Top_10](https://www.owasp.org/images/5/51/OWASP_Top_10_2013-Chinese-V1.2.pdf)之类的文章来完善防护细节。

### 性能
作为中间代理服务器，性能肯定是大多数人会首先关心的，所以我做了一个基准测试，具体的测试数据见[GitHub](https://github.com/chengdedeng/waf#性能)。从测试数据可以看出，Java的Proxy跟Nginx比，
的确有不小的差距，但是我觉得作为一个无状态的中间件，只要性能不是太差同时能够水平扩展，那么这个中间件就是可以被接受的。enjoy yourself!
