title: HttpProxy研发心得
date: 2017-11-13 14:34:00
categories: 技术
tags: [Http,TLS,Socks5,Proxy,ShadowSocks,Netty,LittleProxy]

-----

>笔者之前为了研究HttpProxy，自己用Java造了一个轮子[WAF](https://github.com/chengdedeng/waf)，随着研究的深入，该项目也逐步成为公司的API Gateway之一，在研发过程中收获很多。

### HTTP协议
既然初衷是HttpProxy，那么HTTP(RFC7230-7235/7216)协议是必然需要了解，这是每个开发人员都或多或少了解的协议。我之前自认为对HTTP还是有足够的了解的，但是经过此次实际运用才发现还有很多细节理解的不够。

1. Header头不区分大小写，而我之前所有的获取header都是一律大写，导致在不同的浏览器表现不一致，有些头拿不到。
2. Http Method了解不够，我们跨域方案采用的是CORS，所以会有OPTIONS请求去询问后端是否支持跨域。用户登录成功之后会返回一个Token，后续的访问都需要在HEAD中带上该标识，由于发起OPTIONS请求是浏览器的自带行为不是XHR请求，所以没有Token。Gateway对所有的请求都会进行拦截，自然这个OPTIONS就不幸命中，解决方法就是本地Cookie也写一份或者OPTIONS请求直接放行。
3. X-Forwarded-For，X-Real-IP使用场景和限制没有充分弄清楚。

<!--more-->

### MITM/TLS
JSSE(Java Secure Socket Extension)之前只是略有接触没有详细研究，为了让Gateway支持TLS/MITM，花了我较多的精力。

1. JDK7/JDK8 JCE的标准不一样，可以参见[链接](https://www.java.com/zh_CN/download/faq/release_changes.xml)，所以导致加密套件和加密长度的不同，反正一句话就是建议大家别用JDK7就对了。
2. 自签名证书和CA签发证书，对于浏览器来说问题都不大，最多就是信任然后就可以继续访问，但是对于后端代码来说不太一样。由于CA的证书是受信任的，但是自签名的却不受代码信任，所以代码中我们需要自定义TrustManager，相当于需要将用户在浏览器中手动点击信任的动作用代码给自动化。
3. MITM中间人拦截，其实就是支持TLS的代理服务器，但是得注意浏览器是不支持开启了HSTS的网站，这个花了我很长时间，开始我以为是不支持开启了HTTP/2的网站，这个我还不敢完全肯定，因为www.jd.com是可以的，但是www.v2ex.com却不幸。支持和不支持的见下面两张图。我刚开始是想用TLS代理来翻墙玩玩，但是其效果非常差，一是加解密非常耗时，二是流量特征明显，不过作为HTTPS抓包工具还是有点用处的。
	![MITM1](/images/MITM1.png)
	![MITM2](/images/MITM2.png)

### ShadowSocks/Socks5
由于业务需求，国内需要访问Amazon和Google的SaaS服务，ShadowSocks这是毫无疑问的选择，目标是国内的程序可以通过HTTP的形式访问国外的SaaS服务，该问题困扰了我好几天。
1. Socks5的握手原理不清晰导致Proxy到目标机的Channel创建之后，我才在Pipeline中加入了Socks5ProxyHandler，也就是说握手已经完成，我把Socks Server当成了下游的Http Proxy，也是由于LittleProxy有ProxyChain的原因对我产生了误导。
2. Wireshark中抓包发现Tcp Window Update，这个会出现在Socks握手之时，当时认为ACK的序号不对，其实是Len=0的包也要加一，对TCP协议的理解还需要加强，下面两张图详细阐述了SS的握手过程。

	![TcpWindowUpdate](/images/TcpWindowUpdate.png)
	![socks5handshake](/images/socks5handshake.png)
