title: Classloader之三方库class overriding
date: 2018-02-25 20:00:00
categories: 技术
tags: [Classloader,Springboot,JSW]

-----
### 上下文
假如我们项目中引入的三方包出现了一个小的bug，我们通常会fixed它，然后创建一个pull request，但是通常我们等不到pull request被merge之后的release版本。
这种现象非常普遍，对于这种情况，我们通过有如下三种方案：
1. 使用merge了pull request的snapshot版本。
2. 自己fork一个分支，发布内部版本。
3. Class Overriding，重写之后的代码就放到自己的项目中。

方案一会引入了很多不稳定的因素到项目之中，方案二自己维护一个临时的稳定分支成本太高，所以它们都不是非常好的方案。方案三则避免了前两种方案的不足，并且等官方release版本发布之后，可以零成本的切换到官方版本。

### 问题
Class Overriding之后，在Classpath中会存在两个同名的该类，一个是位于自己项目中另一个是三方库中的原有的。那么问题就来，Classloader究竟会加载谁呢？
Classloader Hierarchy大家都比较熟悉，网上也有大量的文章来阐述。我们不但要清楚parent delegation model，而且需要明白`相同的Class被不同的Classloader加载，在JVM中它们也是不同的Class`。显然此处这并不是我们想要分析的重点，除非我们bug刚好出现在Custemclassloader，也就是说我们要分析的是同一个Classloader对Classpath下名称相同但位置不同的Class资源的加载顺序。

<!--more-->

### 分析
要分析该问题，我们需要给JVM配置上**-verbose:class**，从而将Class的加载信息打印出来。我们需要分为四种场景来分析：
1. IDE环境
2. Fat Jar（Spring Boot）
3. War
4. Jar/JSW（Java Service Wrapper）

为了测试上述的五种情况，我分别用[WAF](https://github.com/chengdedeng/waf)和[spring-boot-loader-play](https://github.com/chengdedeng/spring-boot-loader-play)两份代码来进行测试。WAF中使用了[LitteProxy](https://github.com/adamfisk/LittleProxy)，由于**org.littleshoot.proxy.impl.ProxyToServerConnection**在连接Socks5 Server存在一个bug，正好可以用来作为测试，该测试代码可以覆盖场景1和场景4。spring-boot-loader-play其实是文章[Spring Boot Classloader and Class Overriding](https://dzone.com/articles/spring-boot-classloader-and-class-override)的测试代码，它可以覆盖场景2。有1、2、4的分析，场景3其实就可以不用测试了，如果感兴趣，自己动手试试就知道了。

### 测试
#### 场景1

```
[Loaded org.littleshoot.proxy.impl.ProxyToServerConnection from file:/Users/guo/work/code/waf/target/classes/]
[Loaded org.littleshoot.proxy.impl.ProxyToServerConnection$1 from file:/Users/guo/work/code/waf/target/classes/]
[Loaded org.littleshoot.proxy.impl.ProxyToServerConnection$2 from file:/Users/guo/work/code/waf/target/classes/]
[Loaded org.littleshoot.proxy.impl.ProxyToServerConnection$3 from file:/Users/guo/work/code/waf/target/classes/]
[Loaded org.littleshoot.proxy.impl.ProxyToServerConnection$4 from file:/Users/guo/work/code/waf/target/classes/]
[Loaded org.littleshoot.proxy.impl.ProxyConnection$ResponseReadMonitor from file:/Users/guo/.m2/repository/org/littleshoot/littleproxy/1.1.2/littleproxy-1.1.2.jar]
[Loaded org.littleshoot.proxy.impl.ProxyToServerConnection$5 from file:/Users/guo/work/code/waf/target/classes/]
[Loaded org.littleshoot.proxy.impl.ProxyToServerConnection$6 from file:/Users/guo/work/code/waf/target/classes/]
[Loaded org.littleshoot.proxy.impl.ProxyConnection$RequestWrittenMonitor from file:/Users/guo/.m2/repository/org/littleshoot/littleproxy/1.1.2/littleproxy-1.1.2.jar]
[Loaded org.littleshoot.proxy.impl.ProxyToServerConnection$7 from file:/Users/guo/work/code/waf/target/classes/]
```

#### 场景4
##### classpath中waf-1.0-SNAPSHOT.jar在littleproxy-1.1.2.jar之前的启动命令

```
/usr/bin/java -server -Xms128m -Xmx128m -Xmn60m -XX:+UseG1GC -Xloggc:/tmp/log/gc.log -XX:+HeapDumpOnOutOfMemoryError -XX:+PrintGCDetails -XX:+PrintGCTimeStamps -verbose:class -Djava.library.path=lib -classpath lib/wrapper.jar:conf:lib/waf-1.0-SNAPSHOT.jar:lib/littleproxy-1.1.2.jar:lib/commons-lang3-3.5.jar:lib/barchart-udt-bundle-2.3.0.jar:lib/netty-all-4.1.17.Final.jar:lib/mitm-2.1.4.jar:lib/bcprov-jdk15on-1.56.jar:lib/bcpkix-jdk15on-1.56.jar:lib/spring-context-4.2.5.RELEASE.jar:lib/spring-aop-4.2.5.RELEASE.jar:lib/aopalliance-1.0.jar:lib/spring-beans-4.2.5.RELEASE.jar:lib/spring-core-4.2.5.RELEASE.jar:lib/spring-expression-4.2.5.RELEASE.jar:lib/slf4j-api-1.7.21.jar:lib/slf4j-log4j12-1.7.7.jar:lib/log4j-1.2.17.jar:lib/metrics-core-3.1.2.jar:lib/metrics-graphite-3.1.2.jar:lib/metrics-log4j-3.1.2.jar:lib/metrics-jvm-3.1.2.jar:lib/metrics-spring-3.1.3.jar:lib/metrics-healthchecks-3.1.2.jar:lib/metrics-annotation-3.1.2.jar:lib/spring-context-support-4.1.6.RELEASE.jar:lib/esapi-2.1.0.1.jar:lib/commons-configuration-1.10.jar:lib/commons-lang-2.6.jar:lib/commons-beanutils-core-1.8.3.jar:lib/commons-fileupload-1.3.1.jar:lib/commons-io-2.2.jar:lib/commons-collections-3.2.2.jar:lib/xom-1.2.5.jar:lib/xml-apis-1.3.03.jar:lib/xercesImpl-2.8.0.jar:lib/xalan-2.7.0.jar:lib/bsh-core-2.0b4.jar:lib/antisamy-1.5.3.jar:lib/nekohtml-1.9.16.jar:lib/commons-httpclient-3.1.jar:lib/batik-css-1.8.jar:lib/batik-ext-1.8.jar:lib/batik-util-1.8.jar:lib/xml-apis-ext-1.3.04.jar:lib/guava-20.0.jar:lib/httpclient-4.5.3.jar:lib/commons-logging-1.2.jar:lib/commons-codec-1.9.jar:lib/httpcore-4.4.6.jar:lib/httpmime-4.5.2.jar:lib/joda-time-2.9.9.jar -Dwrapper.key=SDxB55oX2r5RDERn -Dwrapper.port=32000 -Dwrapper.jvm.port.min=31000 -Dwrapper.jvm.port.max=31999 -Dwrapper.pid=27529 -Dwrapper.version=3.2.3 -Dwrapper.native_library=wrapper -Dwrapper.service=TRUE -Dwrapper.cpu.timeout=10 -Dwrapper.jvmid=1 org.tanukisoftware.wrapper.WrapperSimpleApp info.yangguo.waf.Application
```
```
INFO   | jvm 1    | 2018/02/26 16:42:23 | [Loaded org.littleshoot.proxy.impl.ProxyToServerConnection from file:/Users/guo/work/code/waf/target/waf-1.0-SNAPSHOT/lib/waf-1.0-SNAPSHOT.jar]
INFO   | jvm 1    | 2018/02/26 16:42:23 | [Loaded org.littleshoot.proxy.impl.ProxyToServerConnection$1 from file:/Users/guo/work/code/waf/target/waf-1.0-SNAPSHOT/lib/waf-1.0-SNAPSHOT.jar]
INFO   | jvm 1    | 2018/02/26 16:42:23 | [Loaded org.littleshoot.proxy.impl.ProxyToServerConnection$2 from file:/Users/guo/work/code/waf/target/waf-1.0-SNAPSHOT/lib/waf-1.0-SNAPSHOT.jar]
INFO   | jvm 1    | 2018/02/26 16:42:23 | [Loaded org.littleshoot.proxy.impl.ProxyToServerConnection$3 from file:/Users/guo/work/code/waf/target/waf-1.0-SNAPSHOT/lib/waf-1.0-SNAPSHOT.jar]
INFO   | jvm 1    | 2018/02/26 16:42:23 | [Loaded org.littleshoot.proxy.impl.ProxyToServerConnection$4 from file:/Users/guo/work/code/waf/target/waf-1.0-SNAPSHOT/lib/waf-1.0-SNAPSHOT.jar]
INFO   | jvm 1    | 2018/02/26 16:42:23 | [Loaded org.littleshoot.proxy.impl.ProxyConnection$ResponseReadMonitor from file:/Users/guo/work/code/waf/target/waf-1.0-SNAPSHOT/lib/littleproxy-1.1.2.jar]
INFO   | jvm 1    | 2018/02/26 16:42:23 | [Loaded org.littleshoot.proxy.impl.ProxyToServerConnection$5 from file:/Users/guo/work/code/waf/target/waf-1.0-SNAPSHOT/lib/waf-1.0-SNAPSHOT.jar]
INFO   | jvm 1    | 2018/02/26 16:42:23 | [Loaded org.littleshoot.proxy.impl.ProxyToServerConnection$6 from file:/Users/guo/work/code/waf/target/waf-1.0-SNAPSHOT/lib/waf-1.0-SNAPSHOT.jar]
INFO   | jvm 1    | 2018/02/26 16:42:23 | [Loaded org.littleshoot.proxy.impl.ProxyConnection$RequestWrittenMonitor from file:/Users/guo/work/code/waf/target/waf-1.0-SNAPSHOT/lib/littleproxy-1.1.2.jar]
INFO   | jvm 1    | 2018/02/26 16:42:23 | [Loaded org.littleshoot.proxy.impl.ProxyToServerConnection$7 from file:/Users/guo/work/code/waf/target/waf-1.0-SNAPSHOT/lib/waf-1.0-SNAPSHOT.jar]
```

##### classpath中waf-1.0-SNAPSHOT.jar在littleproxy-1.1.2.jar之后的启动命令

```
/usr/bin/java -server -Xms128m -Xmx128m -Xmn60m -XX:+UseG1GC -Xloggc:/tmp/log/gc.log -XX:+HeapDumpOnOutOfMemoryError -XX:+PrintGCDetails -XX:+PrintGCTimeStamps -verbose:class -Djava.library.path=lib -classpath lib/wrapper.jar:conf:lib/littleproxy-1.1.2.jar:lib/waf-1.0-SNAPSHOT.jar:lib/commons-lang3-3.5.jar:lib/barchart-udt-bundle-2.3.0.jar:lib/netty-all-4.1.17.Final.jar:lib/mitm-2.1.4.jar:lib/bcprov-jdk15on-1.56.jar:lib/bcpkix-jdk15on-1.56.jar:lib/spring-context-4.2.5.RELEASE.jar:lib/spring-aop-4.2.5.RELEASE.jar:lib/aopalliance-1.0.jar:lib/spring-beans-4.2.5.RELEASE.jar:lib/spring-core-4.2.5.RELEASE.jar:lib/spring-expression-4.2.5.RELEASE.jar:lib/slf4j-api-1.7.21.jar:lib/slf4j-log4j12-1.7.7.jar:lib/log4j-1.2.17.jar:lib/metrics-core-3.1.2.jar:lib/metrics-graphite-3.1.2.jar:lib/metrics-log4j-3.1.2.jar:lib/metrics-jvm-3.1.2.jar:lib/metrics-spring-3.1.3.jar:lib/metrics-healthchecks-3.1.2.jar:lib/metrics-annotation-3.1.2.jar:lib/spring-context-support-4.1.6.RELEASE.jar:lib/esapi-2.1.0.1.jar:lib/commons-configuration-1.10.jar:lib/commons-lang-2.6.jar:lib/commons-beanutils-core-1.8.3.jar:lib/commons-fileupload-1.3.1.jar:lib/commons-io-2.2.jar:lib/commons-collections-3.2.2.jar:lib/xom-1.2.5.jar:lib/xml-apis-1.3.03.jar:lib/xercesImpl-2.8.0.jar:lib/xalan-2.7.0.jar:lib/bsh-core-2.0b4.jar:lib/antisamy-1.5.3.jar:lib/nekohtml-1.9.16.jar:lib/commons-httpclient-3.1.jar:lib/batik-css-1.8.jar:lib/batik-ext-1.8.jar:lib/batik-util-1.8.jar:lib/xml-apis-ext-1.3.04.jar:lib/guava-20.0.jar:lib/httpclient-4.5.3.jar:lib/commons-logging-1.2.jar:lib/commons-codec-1.9.jar:lib/httpcore-4.4.6.jar:lib/httpmime-4.5.2.jar:lib/joda-time-2.9.9.jar -Dwrapper.key=Q2nBncFVdPijJ62c -Dwrapper.port=32000 -Dwrapper.jvm.port.min=31000 -Dwrapper.jvm.port.max=31999 -Dwrapper.pid=27838 -Dwrapper.version=3.2.3 -Dwrapper.native_library=wrapper -Dwrapper.service=TRUE -Dwrapper.cpu.timeout=10 -Dwrapper.jvmid=1 org.tanukisoftware.wrapper.WrapperSimpleApp info.yangguo.waf.Application
```

```
INFO   | jvm 1    | 2018/02/26 17:07:01 | [Loaded org.littleshoot.proxy.impl.ProxyToServerConnection from file:/Users/guo/work/code/waf/target/waf-1.0-SNAPSHOT/lib/littleproxy-1.1.2.jar]
INFO   | jvm 1    | 2018/02/26 17:07:01 | [Loaded org.littleshoot.proxy.impl.ProxyToServerConnection$1 from file:/Users/guo/work/code/waf/target/waf-1.0-SNAPSHOT/lib/littleproxy-1.1.2.jar]
INFO   | jvm 1    | 2018/02/26 17:07:01 | [Loaded org.littleshoot.proxy.impl.ProxyToServerConnection$2 from file:/Users/guo/work/code/waf/target/waf-1.0-SNAPSHOT/lib/littleproxy-1.1.2.jar]
INFO   | jvm 1    | 2018/02/26 17:07:01 | [Loaded org.littleshoot.proxy.impl.ProxyToServerConnection$3 from file:/Users/guo/work/code/waf/target/waf-1.0-SNAPSHOT/lib/littleproxy-1.1.2.jar]
INFO   | jvm 1    | 2018/02/26 17:07:01 | [Loaded org.littleshoot.proxy.impl.ProxyToServerConnection$4 from file:/Users/guo/work/code/waf/target/waf-1.0-SNAPSHOT/lib/littleproxy-1.1.2.jar]
INFO   | jvm 1    | 2018/02/26 17:07:01 | [Loaded org.littleshoot.proxy.impl.ProxyConnection$ResponseReadMonitor from file:/Users/guo/work/code/waf/target/waf-1.0-SNAPSHOT/lib/littleproxy-1.1.2.jar]
INFO   | jvm 1    | 2018/02/26 17:07:01 | [Loaded org.littleshoot.proxy.impl.ProxyToServerConnection$5 from file:/Users/guo/work/code/waf/target/waf-1.0-SNAPSHOT/lib/littleproxy-1.1.2.jar]
INFO   | jvm 1    | 2018/02/26 17:07:01 | [Loaded org.littleshoot.proxy.impl.ProxyToServerConnection$6 from file:/Users/guo/work/code/waf/target/waf-1.0-SNAPSHOT/lib/littleproxy-1.1.2.jar]
INFO   | jvm 1    | 2018/02/26 17:07:01 | [Loaded org.littleshoot.proxy.impl.ProxyConnection$RequestWrittenMonitor from file:/Users/guo/work/code/waf/target/waf-1.0-SNAPSHOT/lib/littleproxy-1.1.2.jar]
INFO   | jvm 1    | 2018/02/26 17:07:01 | [Loaded org.littleshoot.proxy.impl.ProxyToServerConnection$7 from file:/Users/guo/work/code/waf/target/waf-1.0-SNAPSHOT/lib/littleproxy-1.1.2.jar]
INFO   | jvm 1    | 2018/02/26 17:07:01 | [Loaded com.google.common.net.HostAndPort from file:/Users/guo/work/code/waf/target/waf-1.0-SNAPSHOT/lib/guava-20.0.jar]
```

启动命令中jar的顺序控制，如果是手写命令很简单，想怎么写都行，JSW包中只需要改conf/wrapper.conf文件中wrapper.java.classpath.`n`，n就是顺序，所以改动更加方便。

#### 场景2
fat jar有多种格式，此处我们只分析一下springboot fat jar，这是最复杂的一种情况。文章[Spring Boot Classloader and Class Overriding](https://dzone.com/articles/spring-boot-classloader-and-class-override)花了大篇幅来阐述其中的门门道道。一切都是由于Springboot项目打的fat包不是一个标准的jar包，对于非标准的jar来说，Class的加载肯定是需要自定义Classloader的，Springboot就是通过[LaunchedURLClassLoader](https://github.com/spring-projects/spring-boot/blob/master/spring-boot-project/spring-boot-tools/spring-boot-loader/src/main/java/org/springframework/boot/loader/LaunchedURLClassLoader.java)来解决该问题。正是由于该LaunchedURLClassLoader的引入，导致复杂度提升。作者使用gradle+springboot plugin来构建，不但麻烦而且最后的方案还是一个半成品，我又使用maven+jsw来构建，则可以完全避开作者文章中的各种麻烦，详见[classloadertest](https://github.com/chengdedeng/classloadertest)。所以很多时候换一种思路，则是柳暗花明。


### 总结
1. IDE环境下，Classloader优先加载本机编译路径下的class。
2. 如果是普通jar包，class的加载顺序就是classpath下资源给出的顺序。在顺序的控制上，可以自己写脚本把需要优先加载的放在前面，而jsw会自动把当前项目的jar放在第一位，所以极其方便。
3. War包WEB-INF/classes下的资源优先于lib中的jar加载。
4. JSW是我认为最优雅。
