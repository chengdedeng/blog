title: 外部HTTP服务访问慢的Case
date: 2015-6-30 18:32:29
categories: 技术
tags: [HTTP]
----

### 问题现象
最近由于业务需求，重写了一套访问中航信IBE+的代理层。由于机票业务分为国内和国际两部分，这两部分都需要访问IBE+，之前是各自为战，各自都有一套访问逻辑。这样做的后果就是在流控(`IBE+对QPS有一定的限制`)和接口管理方面得不到很好的控制，因此才写了这样的一个代理层。这样的设计其实没有任何问题，在微服务和SOA大行其道的今天这都是主流设计思路。该系统很快便上线了，但是很快我就发现IBE+的服务访问都非常慢，因为之前我没有涉及到这块，我就问了下之前负责这块的同事，他们说的确比较慢，我也就没有太上心。突然有天我打开生产环境的监控，我便发现了问题，大概30秒左右便有一次非常慢的操作。关于30秒的问题，之前我通过tcpdump抓包分析过，以为是Http Basic鉴权导致的，在新写的代理层中已经对此处进行了优化，但是结果看来不是很理想，然后便有了这个Case。

<!--more-->

-----------

### 问题定位

#### 猜测+初步验证
30秒左右就会慢一次，思来想去就只有DNS的问题，因为Java的DNS默认缓存大概就是30秒。然后我迅速写了如下的一段测试代码：

```java
import java.net.Inet4Address;
import java.net.InetAddress;
import java.net.UnknownHostException;
import java.util.Date;

/**
 * Created by IntelliJ IDEA
 * User:杨果
 * Date:15/6/18
 * Time:下午2:55
 * <p/>
 * Description:DNS解析速度测试
 */

public class Test {
    public static void main(String args[]) throws UnknownHostException, InterruptedException {
        for (int i = 0; i < 10; i++) {
            String host = "agibe.travelsky.com";

            long begin1 = new Date().getTime();
            Inet4Address.getByName(host);
            long end1 = new Date().getTime();
            System.out.println(end1 - begin1);


            long begin2 = new Date().getTime();
            InetAddress.getAllByName(host);
            long end2 = new Date().getTime();
            System.out.println(end2 - begin2);

			System.out.println("----");
            Thread.sleep(1000*5);
        }
    }
}
```
放到生产环境的服务器一测，果然每个30秒便巨慢一次，大概每次解析在5秒钟左右。网上还有一套通过反射来获取Java DNS缓存时间的测试代码，代码非常简单如下所示：

```java
import java.lang.reflect.Field;
import java.net.InetAddress;
import java.text.SimpleDateFormat;
import java.util.Date;
import java.util.Map;

/**
 * Created by IntelliJ IDEA
 * User:杨果
 * Date:15/5/7
 * Time:下午5:10
 * <p/>
 * Description:
 * <p/>
 * dns缓存时间测试
 */
public class DnsTester {
    public static void main(String[] args) throws Exception {
        System.out.println("start loop\n\n");
        for (int i = 0; i < 30; ++i) {
            Date d = new Date();
            SimpleDateFormat sdf = new SimpleDateFormat("yyyy-MM-dd HH:mm:ss");
            System.out.println("current time:" + sdf.format(d));
            InetAddress addr1 = InetAddress.getByName("www.baidu.com");
            String addressCache = "addressCache";
            System.out.println(addressCache);
            printDNSCache(addressCache);
            System.out.println("getHostAddress:" + addr1.getHostAddress());
            System.out.println("*******************************************");
            System.out.println("\n");
            java.lang.Thread.sleep(10000);
        }

        System.out.println("end loop");
    }


    private static void printDNSCache(String cacheName) throws Exception {
        Class<InetAddress> klass = InetAddress.class;
        Field acf = klass.getDeclaredField(cacheName);
        acf.setAccessible(true);
        Object addressCache = acf.get(null);
        Class cacheKlass = addressCache.getClass();
        Field cf = cacheKlass.getDeclaredField("cache");
        cf.setAccessible(true);
        Map<String, Object> cache = (Map<String, Object>) cf.get(addressCache);
        for (Map.Entry<String, Object> hi : cache.entrySet()) {
            Object cacheEntry = hi.getValue();
            Class cacheEntryKlass = cacheEntry.getClass();
            Field expf = cacheEntryKlass.getDeclaredField("expiration");
            expf.setAccessible(true);
            long expires = (Long) expf.get(cacheEntry);

            System.out.println(hi.getKey() + " " + new Date(expires) + " " + hi.getValue());
        }
    }
}
```

#### 深入代码分析+测试
经过初步的分析和测试，我已经找到问题的所在，现在就要深入代码进行分析看看是否跟我的猜想吻合。由于我的代码使用的是HTTP Client4.3.2，没有配置**DnsResolver**，那么肯定使用的是**SystemDefaultDnsResolver**。这点确认之后，便可以证明第一步的验证逻辑没有偏离方向。然后我便写了如下的一段**Btrace**脚本，看看线上的服务究竟慢在哪里，脚本如下：

```java
@BTrace
public class HttpClientSystemDefaultDnsResolverTracer {
    @TLS
    static long beginTime;

    @OnMethod(
            clazz = "org.apache.http.impl.conn.SystemDefaultDnsResolver",
            method = "resolve"
    )
    public static void traceGetByNameBegin() {
        beginTime = timeMillis();
    }

    @OnMethod(
            clazz = "org.apache.http.impl.conn.SystemDefaultDnsResolver",
            method = "resolve",
            location = @Location(Kind.RETURN)
    )
    public static void traceGetByNameEnd() {
        println(strcat(strcat("org.apache.http.impl.conn.SystemDefaultDnsResolver.resolve 耗时:", str(timeMillis() - beginTime)), "ms"));
    }
}
```
虽然**BTrace**比较安全，但是我将一台机器的流量切走了大部分，只留下很少的部分来做测试，测试下来果然跟我第一步吻合。

-----

### 寻找修复方案

做了上面充足的测试之后，我拿着手上的数据找到复杂网络的同学，开始跟他定位问题。当我看见**/etc/resolv.conf**下面的**nameserver 114.114.114.114**的时候，我心中一震。我马上让他切换到我们的DNS之后，再一运行第一步的测试代码，速度果然马上提升了好几个数量级。我心中一喜，马上让他将机器的DNS切成我们自己的DNS。然后开始开始观察，经过两个小时之后，我发现速度还是没有提升起来。我就开始纳闷了，这是什么问题呢，我们问题不是已经定位到了嘛？我开始怀疑是不是进程没有重启的原因，我马上找了一台没有修改成我们自己DNS的机器将第一个程序跑起来。然后我开始修改DNS，果然修改之后程序的解析速度没有提升，重启进程问题解决。

看着**QPS**的缓慢爬升以及**Average Response Time**的缓慢下降，心中暗爽。你可能以为这个Case已经结束，的确当时我也是这样认为的。

----

### DNS解析真的慢？
大概过了两天，我突然觉得意识到，在这个Case中我遗漏掉了什么。114真的有这慢吗？虽然作为一个经常被吐槽最容易被污染DNS，114的速度我记得还是相当靠谱的啊。Google了一圈，在Stack OverFlow上找到一些讨论，说是IPV6的原因导致的。

我马上进行了尝试，首先我不指定ipv4，例子如下：

```bash
[root@centos87 ~]# wget agibe.travelsky.com
--2015-07-01 17:56:13--  http://agibe.travelsky.com/
Resolving agibe.travelsky.com... 122.119.122.38
Connecting to agibe.travelsky.com|122.119.122.38|:80... connected.
HTTP request sent, awaiting response... 200 OK
Length: 239 [text/html]
Saving to: “index.html.3”

100%[=============================================================================================>] 239         --.-K/s   in 0s

2015-07-01 17:56:18 (21.4 MB/s) - “index.html.3” saved [239/239]

```

然后我再指定ipv4试了一把，如下：

```bash
root@centos87 ~]# wget -4 agibe.travelsky.com
--2015-07-01 17:57:20--  http://agibe.travelsky.com/
Resolving agibe.travelsky.com... 122.119.122.38
Connecting to agibe.travelsky.com|122.119.122.38|:80... connected.
HTTP request sent, awaiting response... 200 OK
Length: 239 [text/html]
Saving to: “index.html.4”

100%[=============================================================================================>] 239         --.-K/s   in 0s

2015-07-01 17:57:20 (21.3 MB/s) - “index.html.4” saved [239/239]
```
结果相当清晰，罪魁祸首就是ipv6。那为啥我换成我们自己的DNS就好了，我又跟负责网络的同学进行了沟通，原来他在DNS服务器上将IPV6的转发进行了关闭，所以可以很快返回。为了证明我的猜想是正确的，我又找了一台DNS为修改的服务器，让运维的同学将ipv6给禁用，经过测试果然奏效。至于如何关闭ipv6，网上有太多教程了。我在测试时候用了下面的方式，个人觉得太暴力，不推荐使用，不过应急可以试试。

```bash
echo 1 > /proc/sys/net/ipv6/conf/all/disable_ipv6
```

------------------

### Java  DNS Lookup
上面通过DNS和系统层面都可以解决该问题，现在便到我们代码层面了。我们知道并非所有操作系统都支持 ipv6 协议，Java networking stack优先尝试检测ipv6 。当然你也可以利用系统属性禁用它。

#### 解析步骤如下：

1. Java networking stack 先确认底层操作系统是否支持ipv6。如果支持ipv6，Java将尝试使用ipv6 stack。
2. 在双堆栈（dual-stack，指ipv4 stack+ipv6 stack）系统上，将创建一个ipv6 socket。在 separate-stack 系统上，事情要复杂得多，Java 将创建两个socket，一个给ipv4 一个给ipv6。
3. 对于客户端TCP应用，一旦socket连上了，那 internet-protocol family type 就固定了，多余的那个socket就关闭了。对于服务器端TCP应用，由于不知道下一个客户端请求用什么ip family type，所以这两个 sockets 将继续保留。对于UDP应用，这两个sockets始终都需要保留。


#### java ipv6 相关的系统参数

1. 首选的协议栈：ipv4还是ipv6；
2. 首选的地址族（address family type）：inet4 还是 inet6。

由于在一个双堆栈系统上，ipv6 socket 能与 ipv4 和 ipv6 对端交互，所以 ipv6 stack 是默认首选项。你可以通过如下系统参数修改配置：

```
java.net.preferIPv4Stack=<true|false>
```
对应的 java 代码是：

```java
java.lang.System.setProperty("java.net.preferIPv4Stack", "true");
```

默认我们首选 ipv4 地址族，也可以通过如下系统参数修改配置：

```
java.net.preferIPv6Addresses=<true|false>
```
对应的 java 代码是：

```java
java.lang.System.setProperty("java.net.preferIPv6Addresses", "false");
```




然后我又用下面的代码进行了尝试，代码如下：

```java
import java.net.Inet4Address;
import java.net.InetAddress;
import java.net.UnknownHostException;
import java.util.Date;

/**
 * Created by IntelliJ IDEA
 * User:杨果
 * Date:15/6/18
 * Time:下午2:55
 * <p/>
 * Description:IPV4 DNS解析速度测试
 */
public class Test {
    public static void main(String args[]) throws UnknownHostException, InterruptedException {
        java.lang.System.setProperty("java.net.preferIPv4Stack", "true");
        java.lang.System.setProperty("java.net.preferIPv6Addresses", "false");
        for (int i = 0; i < 10; i++) {
            String host = "agibe.travelsky.com";

            long begin1 = new Date().getTime();
            Inet4Address.getByName(host);
            long end1 = new Date().getTime();
            System.out.println(end1 - begin1);


            long begin2 = new Date().getTime();
            InetAddress.getAllByName(host);
            long end2 = new Date().getTime();
            System.out.println(end2 - begin2);

			System.out.println("----");
            Thread.sleep(1000*5);
        }
    }
}
```
-------------
### 结论
>看似简单的结论背后，如果去深挖，总会发现很多有趣的东西，enjoy yourself。
