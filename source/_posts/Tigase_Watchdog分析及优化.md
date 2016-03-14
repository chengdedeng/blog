title: Tigase Watchdog分析及优化
date: 2014-9-10 23:14:34
categories: 技术
tags: [Java,IM]
----

>**TCP链接回收**是所有长连接服务器必须要面对的问题，我前面的有些文章其实已经涉及到该方面的知识。但是本篇文章主要分析和讲述tigase在移动网络中，链接的回收原理及优化方案。

### 心跳
心跳的作用是告诉对方，自己还活着；当然有人说还可以判断链路是否可到达，但是我觉得在JAVA层面这个不是很靠谱。因为当数据刷新到TCP的缓冲区之后，就成功返回了。这也正是即使使用TCP，还需要在业务层封装各种ACK包的原因，即使TCP是可靠链接，它的可靠我理解为是传输时可靠，而非传输前往buffer中写数据的可靠。

在开始之前，首先明确两个概念。

```
1. S2S:服务器与服务器之间的通信，网络稳定，基本在同一个IDC内部，当然也有可能跨IDC。
2. C2S:移动设备与服务器之间的通信，网络不稳定。
```

<!--more-->

---------

### 问题
同事在测试时发现kill -9或者直接将测试机断网之后，发现大量的established connections没有回收，也就是说session的回收出现问题。
### 定位
第一反应，问题肯定是出现在__Watchdog__了，直接上代码：


``` java
 	/**
     * Looks in all established connections and checks whether any of them is
     * dead....
     */
    private class Watchdog
            implements Runnable {
        /**
         * Method description
         */
        @Override
        public void run() {
            while (true) {
                try {

                    // Sleep...
                    //----------作者的原代码
                    //Thread.sleep(10 * MINUTE);
                    //----------作者的原代码
                    //----------修改后的代码
                    //10分钟时间太长了，对于移动网络来说
                    Thread.sleep(30 * SECOND);
                    //----------修改后的代码
                    ++watchdogRuns;

                    // Walk through all connections and check whether they are
                    // really alive...., try to send space for each service which
                    // is inactive for hour or more and close the service
                    // on Exception
                    doForAllServices(new ServiceChecker<IO>() {
                        @Override
                        public void check(final XMPPIOService service) {
                            try {
                                if (null != service) {
                                    long curr_time = System.currentTimeMillis();
                                    long lastTransfer = service.getLastTransferTime();

                                    if (curr_time - lastTransfer >= maxInactivityTime) {

                                        // Stop the service is max keep-alive time is exceeded
                                        // for non-active connections.
                                        if (log.isLoggable(Level.INFO)) {
                                            log.log(Level.INFO,
                                                    "{0}: Max inactive time exceeded, stopping: {1}",
                                                    new Object[]{getName(),
                                                            service});
                                        }
                                        ++watchdogStopped;
                                        service.stop();
                                    } else {
                                        //此处的判断对于c2s来说是无意义的，此处写超时不修改的原因是，ClientConnectionManager中的时间已经修改。对于S2S,BOSH,CLUSTER写是有必要的                                    
                                        if (curr_time - lastTransfer >= (29 * MINUTE)) {

                                            // At least once an hour check if the connection is
                                            // still alive.
                                            service.writeRawData(" ");
                                            ++watchdogTests;
                                        }
                                    }
                                }
                            } catch (Exception e) {

                                // Close the service....
                                try {
                                    if (service != null) {
                                        log.info(getName() + "Found dead connection, stopping: " + service);
                                        ++watchdogStopped;
                                        service.forceStop();
                                    }
                                } catch (Exception ignore) {

                                    // Do nothing here as we expect Exception to be thrown here...
                                }
                            }
                        }
                    });
                } catch (InterruptedException e) {    /* Do nothing here */
                }
            }
        }
    }
}    // ConnectionManager
```
注释我已经写的很清楚，对于不稳定的移动网络来说，10分钟的时间的确太长了。作者的开发当时主要是针对有线网络而设计的，所以此处需要优化。然后注意这行：

``` java
 if (curr_time - lastTransfer >= maxInactivityTime) {
```
马上又能定位到**ClientConnectionManager**的下面方法：

``` java
    @Override
    protected long getMaxInactiveTime() {
        return 3 * MINUTE;
        //-------作者原代码
		//return 24 * HOUR;
		//-------作者原代码
    }
```
此处我修改为了3分钟，因为我们手机端心跳周期是59(质数)秒，我们默认3个周期如果没有心跳上来，就表示链接坏掉了。当然这个地方我为什么只改**ClientConnectionManager**，而不改动Bosh、S2S、WebSocket等，因为这些我们都认为是走有线网络的，比较稳定。

------

### 回收速度
Watchdog只是回收链接，不论是Stop还是ForceStop，都其实都是通过SocketIO/TLSIO/ZLibIO来关闭。全双工的TCP链接已经不存在了，所以会卡在**FIN-WAIT-1**，所以别忘了调整TCP参数。具体可以参考：[Linux TCP参数调整](/2013/10/30/TCP参数调整)
