title: Haproxy之websocket的负载均衡方案
date: 2014-6-25 10:3:30
categories: 技术
tags: [负载均衡]
------

>最近用websokcet写了一套简单的内部聊天服务，我选择了简单易用的haproxy实现负载均衡。

### How does websocket work ?
通常，一个websocket请求的HTTP头如下所示:

```
GET ws://ws.im.yangguo.info/ws HTTP/1.1
Host: test.ws.im.yunma1688.com
Connection: Upgrade
Pragma: no-cache
Cache-Control: no-cache
Upgrade: websocket
Origin: http://short.im.yangguo.info
Sec-WebSocket-Version: 13
DNT: 1
User-Agent: Mozilla/5.0 (Macintosh; Intel Mac OS X 10_10_5) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/47.0.2526.80 Safari/537.36
Accept-Encoding: gzip, deflate, sdch
Accept-Language: zh-CN,zh;q=0.8,en;q=0.6,de-DE;q=0.4,de;q=0.2,zh-TW;q=0.2
Sec-WebSocket-Key: zGYcUVMijj7ihvhLCEegZQ==
Sec-WebSocket-Extensions: permessage-deflate; client_max_window_bits
```
这里最核心的部分就是`Connection: Upgrade`头，它让client端知道server端会改变协议，变成如`Upgrade: websocket`header中所述的协议。

<!--more-->

如果server端有提供websocket协议的能力，那么它就会返回如下的内容:

```
General:
Request URL:ws://ws.im.yangguo.info/ws
Request Method:GET
Status Code:101 Switching Protocols

Response Headers:
view source
Connection:Upgrade
Sec-WebSocket-Accept:DVDuNyg5QVQp78vY6Ts9/uXsAWE=

Upgrade:websocket
```
Status code 101表示协议切换成功(from http to websocket)。

---------

### HAProxy and Websockets
如上所示，websockets嵌入了两种协议：
1. HTTP：在websocket启动时
2. TCP：websocket数据交换

任何时候HAProxy必须在一条没有被打断的TCP链路上支持websockets中的两种协议，这里有两件事情需要注意：
1. 能够从HTTP切换到TCP并且链路不能断开。
2. 同时支持两种协议的超时管理器

幸运的是，HAProxy能够完美的解决上面的两个要求，并且支持多种的websockets负载均衡模式。它不但可以将流量转发到不到的后端机器，并且还可以执行健康检查(仅限链路建立阶段)。


下图详细说明了每个阶段发生了什么和每个阶段的涉及到的timeout：

![timeout_haproxy](/images/timeout_websocket.png)

在链路建立阶段，HAProxy以HTTP模式运行，处理七层的的信息。它会自动检测连接：升级交换，并准备好切换到隧道模式如果升级协商成功。在这个阶段会涉及到3个timeout:

1. client timeout:client端不活跃的时间
2. connect timeout:允许TCP链接建立的时间
3. server timeout:允许server端处理请求的时间

如果一切顺利，websocket建立成功，然后HAProxy故障转义到tunnel模式，此时HTTP层面在没有任何数据传输。因此该阶段只涉及到一个timeout

1. tunnel timeout:优先于client和server端超时

connect timeout将不再使用，因为TCP链路已经建立完成。





#### 调度模式
**Haproxy**，负载均衡调度模式有如下几种:

1. roundrobin，表示简单的轮询，这个不多说，这个是负载均衡基本都具备的；
2. static-rr，表示根据权重;
3. leastconn，表示最少连接者先处理；
4. source，表示根据请求源IP；
5. uri，表示根据请求的URI；
6. url_param，表示根据请求的URl参数'balance url_param' requires an URL parameter name
7. hdr(name)，表示根据HTTP请求头来锁定每一次HTTP请求；
8. rdp-cookie(name)，表示根据据cookie(name)来锁定并哈希每一次TCP请求。


#### 整体配置

```
global
  	log 127.0.0.1 local3    #local3是设备，对应于 /etc/rsyslog.conf中的配置，默认回收info的日志级别
  	daemon
defaults
  	mode http
  	log global
  	option httplog
  	option  http-server-close
	option  dontlognull
  	option  redispatch
  	option  contstats
  	retries 3
  	backlog 10000
  	timeout client          25s
  	timeout connect          5s
  	timeout server          25s
# timeout tunnel available in ALOHA 5.5 or HAProxy 1.5-dev10 and higher
  	timeout tunnel        15m
  	timeout http-keep-alive  1s
  	timeout http-request    15s
  	timeout queue           30s
  	timeout tarpit          60s
  	default-server inter 3s rise 2 fall 3
  	option forwardfor

frontend http
    bind *:80
	## routing based on Host header
 	acl host_ws hdr_beg(Host) -i ws.yangguo.info
  	use_backend bk_ws if host_ws
	## routing based on websocket protocol header
  	acl hdr_connection_upgrade hdr(Connection)  -i upgrade
  	acl hdr_upgrade_websocket  hdr(Upgrade)     -i websocket
  	use_backend bk_ws if hdr_connection_upgrade hdr_upgrade_websocket

	acl host_short hdr_beg(Host) -i short.im.yangguo.info
	use_backend bk_short if host_short

backend bk_ws
    balance roundrobin
    server server1 192.168.199.125:60000 maxconn 10000 weight 10 cookie server1  check
	server server1 192.168.199.126:60000 maxconn 10000 weight 10 cookie server1  check

backend bk_short
    balance roundrobin
    server server1 192.168.199.125:8082 cookie server1 check
    server server1 192.168.199.126:8082 cookie server1 check
```
