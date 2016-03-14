title: Tigase集群方案及配置说明
date: 2014-9-20 22:14:34
categories: 技术
tags: [Java,IM,Tigase]
----
>该文档主要是描述Tigase整体架构和一些配置说明，整体架构我们采用**config-type=—-genconfig-def**的server加**config-type=—-genconfig-comp**的外部component。同时还会对SM的插件和错误代码进行说明，方便大家在开发及配置时参考。由于外部component较多，此处选择了较为复杂的MUC作为配置案例。pubsub，proxy，message-archive，msn，自己开发的componet，配置都的思想都基本一样。但是限于本人认识有限，如果有任何错误或者歧义，请大家及时指正。

### Server(c2s+s2s+sm+ext2s)集群+MUC(comp)集群

#### 架构图

![tigase架构图](/images/ExternalCompClustering.003_0.png)

<!--more-->

#### Server1(hostname: vm-128-157)配置

```
--debug = server,cluster,xmpp.impl
 --virt-hosts = xmpp.dev.pajkdc.com
--user-db-uri = jdbc:mysql://10.0.128.115:3307/tigase?user=pajk&password=qawsed123
--user-db = mysql
--admins = admin@xmpp.dev.pajkdc.com
config-type = --gen-config-def
--cluster-mode = true
--cluster-connect-all = true
--cluster-nodes=vm-128-157:yangguo:5277,vm-128-158:yangguo:5277
--sm-plugins =-starttls,+urn:ietf:params:xml:ns:messageConsumeACK,+urn:ietf:params:xml:ns:xmpp-user,-message-archive-xep-0136,-jabber:iq:register,-presence,-http://jabber.org/protocol/stats,-vcard-temp,-jabber:iq:roster,-pep
--comp-name-1 = ext
--comp-class-1 = tigase.server.ext.ComponentProtocol
--external = muc.xmpp.dev.pajkdc.com:yangguo:listen:5270:vm-128-157:accept:ReceiverBareJidLB
--auth-domain-repo-pool=com.pajk.im.repository.ExtraAuthRepositoryMDImpl
--user-domain-repo-pool=com.pajk.im.repository.ExtraUserRepositoryMDImpl
--comp-name-2 = ChannelMsgExtComponent
--comp-class-2 = com.pajk.im.cmp.ChannelMsgExtComponent
--cm-ht-traffic-throttling = xmpp:250k:0:disc,bin:1024m:0:disc
--cm-traffic-throttling=xmpp:2500:0:disc,bin:10m:0:disc
--new-connections-throttling=5222:500,5223:50,5269:100,5280:1000,5290:100
--net-buff-high-throughput=256k
```
-------

#### Server2(hostname: vm-128-158)配置

```
--debug = server,cluster,xmpp.impl
--virt-hosts = xmpp.dev.pajkdc.com
--user-db-uri = jdbc:mysql://10.0.128.115:3307/tigase?user=pajk&password=qawsed123
--user-db = mysql
--admins = admin@xmpp.dev.pajkdc.com
config-type = --gen-config-def
--cluster-mode = true
--cluster-connect-all = true
--cluster-nodes=vm-128-157:yangguo:5277,vm-128-158:yangguo:5277
--sm-plugins =-starttls,+urn:ietf:params:xml:ns:messageConsumeACK,+urn:ietf:params:xml:ns:xmpp-user,-message-archive-xep-0136,-jabber:iq:register,-presence,-http://jabber.org/protocol/stats,-vcard-temp,-jabber:iq:roster,-pep
--comp-name-1 = ext
--comp-class-1 = tigase.server.ext.ComponentProtocol
--external = muc.xmpp.dev.pajkdc.com:yangguo:listen:5270:vm-128-158:accept:ReceiverBareJidLB
--auth-domain-repo-pool=com.pajk.im.repository.ExtraAuthRepositoryMDImpl
--user-domain-repo-pool=com.pajk.im.repository.ExtraUserRepositoryMDImpl
--comp-name-2 = ChannelMsgExtComponent
--comp-class-2 = com.pajk.im.cmp.ChannelMsgExtComponent
--cm-ht-traffic-throttling = xmpp:250k:0:disc,bin:1024m:0:disc
--cm-traffic-throttling=xmpp:2500:0:disc,bin:10m:0:disc
--new-connections-throttling=5222:500,5223:50,5269:100,5280:1000,5290:100
--net-buff-high-throughput=256k
```
---------

#### muc1配置

```
config-type = --gen-config-comp
--user-db = mysql
--admins = admin@xmpp.dev.pajkdc.com
--user-db-uri = jdbc:mysql://10.0.128.115:3307/tigase?user=pajk&password=qawsed123
--virt-hosts = xmpp.dev.pajkdc.com
--comp-name-1 = muc
--debug = server
--comp-class-1 = tigase.muc.MUCComponent
--external = muc.xmpp.dev.pajkdc.com:yangguo:connect:5270:xmpp.dev.pajkdc.com;vm-128-157;vm-128-158:accept
```

--------

#### muc2配置
```
config-type = --gen-config-comp
--user-db = mysql
--admins = admin@xmpp.dev.pajkdc.com
--user-db-uri = jdbc:mysql://10.0.128.115:3307/tigase?user=pajk&password=qawsed123
--virt-hosts = xmpp.dev.pajkdc.com
--comp-name-1 = muc
--debug = server
--comp-class-1 = tigase.muc.MUCComponent
--external = muc.xmpp.dev.pajkdc.com:yangguo:connect:5270:xmpp.dev.pajkdc.com;vm-128-157;vm-128-158:accept
```

---------

### Tigase MUC思路(引用于Tigase作者Kobit)
> As far as I know there is no such thing like a persistent member of the MUC room. This is not a part of the protocol hence it is not implemented anywhere. Even if we had created something like that in our MUC server it would not work with any XMPP client.
The whole idea behind MUC is to allow for communication between ONLINE users. Therefore, once the user logs off it is removed from the MUC.
However, most of the XMPP clients have a feature to save a MUC bookmark and automatically join the room after login to the XMPP Server.
The MUC room has quite extensive configuration settings, you can list users who are allowed to join the room, you can make the room password protected, you can assign different roles to different users. All of this is supported by our MUC.

-----

`注意事项`:上面的配置，由于是一个完整的分布式环境，所以特别需要注意网络状况。特别是在server端开启本地端口，监听外部component服务链接的时候。尤其是使用无线网络的笔记本电脑，有时候无线网络会导致里面的java代码不能绑定本地端口的情况。面对这种情况，请大家将无线网络断开重连。

----

### Tigase XMPP Server configuration properties

参数|说明|参考
-----|-----|-------|
--admins|管理员账号,管理可以disco插件对server进行管理|
--auth-db|权限认证的db,支持mysql/pgsql/ldap/drupal/tigase-auth/tigase-custom/class name|
--auth-db-uri|权限认证db的uri|
--auth-domain-repo-pool||
--auth-repo-pool||
--auth-repo-pool-size||
--bind-ext-hostnames||
--bosh-close-connection||
--bosh-extra-headers-file||
--cl-conn-repo-class||
--client-access-policy-file||
--cluster-connect-all|动态的将不在集群配置中的节点加入到集群中，默认为false|
--cluster-mode|是否开启集群|
--cluster-nodes|在集群每台机器上确保每个hostname能够正常被DNS解析|[参考](http://www.tigase.org/content/cluster-nodes)
--cm-ht-traffic-throttling|s2s,external component流控|
--cm-see-other-host|xmpp提供的load balance方案|[参考1](http://wiki.jabbercn.org/RFC6120#see-other-host),[参考2](http://www.tigase.org/content/tigase-load-balancing),[参考3](http://www.tigase.org/content/xep-0280-message-carbons)
--cm-traffic-throttling|c2s的流控，每分钟xmpp包为2500个，整个生命周期没有限制;每分钟的流量为20m，整个生命周期没有限制|
--cmpname-ports||
--comp-class|需要加载到server的额外的component，特别是使用了config-type分离了各个component的应用中|[参考](http://www.tigase.org/content/comp-class)
--comp-name|非独立的component(需要被server加载)名称|[参考](http://www.tigase.org/content/comp-name)
--cross-domain-policy-file||
--data-repo-pool-size||
--debug|开启调试模式|[参考](http://www.tigase.org/tigase-debuging)
--debug-packages|输出指定包路径的日志，多个包用逗号分隔|
--domain-filter-policy||
--elements-number-limit||
--ext-comp|弃用，建议使用--external来加载外部component(如:MUC/PubSub/IM transort)，内部component还只能使用这个，如CS和SM分离|[参考](http://www.tigase.org/content/ext-comp)
--extcomp-repo-class||
--external|监听外部component或者将component连接到server|[参考1](http://www.tigase.org/content/external)，[参考2](http://www.tigase.org/content/basic-configuration-options-external-component)
--hardened-mode||
--max-queue-size||
--monitoring||
--net-buff-high-throughput|默认为64k，服务器之间通信如果流量较大，可以适度调大该值，线上调为256k|
--net-buff-standard||
--new-connections-throttling|各个端口每秒新建连接的流控，主要是防止机器重启时，大量连接打死server|
--nonpriority-queue||
--queue-implementation||
--roster-implementation||
--s2s-ejabberd-bug-workaround-active||
--s2s-secret||
--s2s-skip-tls-hostnames||
--script-dir||
--sm-cluster-strategy-class||
--sm-plugins|很多无用的plugin建议不加载，特别是正式环境中|
--sm-threads-pool||
--ssl-certs-location||
--ssl-container-class||
--ssl-def-cert-domain||
--stats-archiv||
--stats-history||
--stringprep-processor||
--test||
--tigase-config-repo-class||
--tigase-config-repo-uri||
--tls-jdk-nss-bug-workaround-active||
--trusted||
--user-db||
--user-db-uri||
--user-domain-repo-pool||
--user-repo-pool||
--user-repo-pool-size||
--vhost-anonymous-enabled||
--vhost-max-users||
--vhost-message-forward-jid||
--vhost-presence-forward-jid||
--vhost-register-enabled||
--vhost-tls-required||
--virt-hosts|虚拟域设置|
--watchdog_delay|心跳检测间隔时间，写检测，默认值10分钟，现在修改为30秒|
--watchdog_ping_type|watchdog ping包类型，写操作，whitespace和xmpp|
--watchdog_timeout|读检测，默认1740000毫秒|
config-type|选择启动的方式，CM和SM分离就是此处配置|[参考](http://www.tigase.org/content/config-type)

----

### sm-plugin说明
参数|说明|参考
-----|-----|-----|
jabber:iq:register|注册服务|
message-archive-xep-0136|消息归档|
jabber:iq:auth|简单用户认证|
urn:ietf:params:xml:ns:xmpp-sasl|SASL协商|[参考](http://wiki.jabbercn.org/RFC6120#SASL.E5.8D.8F.E5.95.86)
urn:ietf:params:xml:ns:xmpp-bind|资源绑定|
urn:ietf:params:xml:ns:xmpp-session|session绑定|
jabber:iq:roster|联系人名单管理|
presence|xmpp顶级元素，上线广播|
jabber:iq:privacy|隐身协议|
jabber:iq:version|客户端版本|
http://jabber.org/protocol/stats|是否发送统计信息，指向jabber.org发送|
startls|tls加密|
msgoffline|离线消息|
vcard-temp|临时的vCard|
http://jabber.org/protocol/commands|管理virtual domains的特别命令|[参考](http://www.tigase.org/content/specification-ad-hoc-commands-used-manage-virtual-domains)
jabber:iq:private|私有数据存储|
urn:xmpp:ping|心跳检测|
pep|发布订阅插件|[参考](http://www.tigase.org/content/extending-pep-plugin)
domain-filter(basic-filter)|domain拦截器|[参考](http://www.tigase.org/content/domain-filter-policy)
amp(basic-filter)|高级消息处理|[参考1](http://wiki.jabbercn.org/XEP-0079)，[参考2](http://www.tigase.org/content/advanced-message-processing-amp-xep-0079)
zlib(basic-filter)|zlib压缩|
message-carbons(basic-filter)|将stanzas投递到用户指定的资源|
disco(basic-filter)|服务发现|

----

### 标准错误代码

代码|说明|
----|----|
302|重定向，尽管HTTP规定中包含八种不同代码来表示重定向，Jabber只用了其中一个（用来代替所有的重定向错误）。不过Jabber代码302是为以后的功能预留的，目前还没有用到。
400|坏请求，Jabber代码400用来通知Jabber客户端，一个请求因为其糟糕的语法不能被识别。例如，当一个Jabber客户端发送一个的订阅请求给它自己活发送一条没有包含“to”属性的消息，Jabber代码400就会产生。
401|未授权的，Jabber代码401用来通知Jabber客户端它们提供的是错误的认证信息，如，在登陆一个Jabber服务器时使用一个错误的密码，或未知的用户名。
402|所需的费用，Jabber代码402为未来使用进行保留，目前还不用到。
403|禁止，Jabber代码403被Jabber服务器用来通知Jabber客户端该客户端的请求可以识别，但服务器拒绝执行。目前只用在注册过程中的密码存储失败。
404|没有找到，Jabber代码404用来表明Jabber服务器找不到任何与JabberID匹配的内容，该JabberID是一个Jabber客户端发送消息的目的地。如，一个用户打算向一个不存在的JabberID发送一条消息。如果接受者的Jabber服务器无法到达，将发送一个来自500级数的错误代码。
405|不允许的，Jabber代码405用在不允许操作被’from’地址标识的JabberID。例如，它可能产生在，一个非管理员用户试图在服务器上发送一条管理员级别的消息，或者一个用户试图发送一台Jabber服务器的时间或版本，或者发送一个不同的JabberID的vCard。
406|不被接受的，Jabber代码406用于服务器因为某些理由不接受一个包。例如，这个可能发生在，一个Jabber客户端试图使用jabber:iq:private在服务器上存储信息，但当前的用于存储的名字空间用”jabber:”开头（在Jabber里是一个被存的XML开头）。另一种可能产生406错误的情况是当一个Jabber客户端试图用一个空密码注册到一台Jabber服务器上。
407|必须注册，Jabber代码407当前不被使用
408|注册超时，当一个Jabber客户端不能在服务器准备好的时间内发起一个请求时，Jabber服务器生成Jabber代码408。这个代码当前只用于Jabber会话管理器使用的零度认证模式中。
409|[冲突](http://wiki.jabbercn.org/RFC6120#.E8.B5.84.E6.BA.90.E7.BB.91.E5.AE.9A)
500|服务器内部错误，当一台Jabber服务器遇到一种预期外的条件，该条件阻止服务器处理来自Jabber客户端的包，这是将用到Jabber代码500。现在，唯一会引发500错误代码的时间是当一个Jabber客户端试图通过服务器认证，而该认证因为某些原因没有被处理（如无法保存密码）。
501|不可执行，当服务器不支持Jabber客户端请求的功能，使用Jabber代码501。例如，该代码只当Jabber客户端发送一个认证请求，而该认证请求不包含服务器配置中定义的任何一种认证方式时，服务器发送Jabber代码501。这个代码还被用于，当一个Jabber客户端试图注册一个不允许注册的服务器。
502|远程服务器错误，当因为无法到达远程服务器导致转发一个包失败时，使用Jabber代码502。该代码发送的特殊例子包括一个远程服务器的连接的失败，无法获取远程服务器的主机名，以及远程服务器错误导致的外部时间过期。
503|服务无法获得，当一个Jabber客户端请求一个服务，而Jabber服务器通常由于一些临时原因无法提供该服务时，使用Jabber代码503。例如，一个Jabber客户端试图发送一条消息给另一个用户，该用户不在线，但它的服务器不提供离线存储服务，服务器将返回一个503错误代码给发送消息的JabberID。当为vcard-temp和jabber:iq:private名字空间设置信息时，出现通过xdb进行数据存储的写入错误，也使用该代码。
504|远程服务器超时，Jabber代码504用于下列情况:试图连接一台服务器发生超时，错误的服务器名。
510|连接失败，Jabber代码510目前还没有使用。

>扩展code(XMPPErrorCodeExtension枚举)，如果大家定义了，请加在此处。
ERROR_TH(4031, "cancel", "登陆过于频繁或者流量过大")

------

### MUC相关技术

* [XEP-0004](http://wiki.jabbercn.org/XEP-0004): 数据表单，用来交换数据。

------

### CS和SM分离

   * tigase中是可行的，但是目前架构没有这样做。这样做会增加网络开销，目前此方案，不是很好，待尝试。

-------

### 使用到得XEP

* [XEP-0184](http://www.xmpp.org/extensions/xep-0184.html): Message Delivery  Receipts，该扩展对server没有任何要求，只要client端支持就行。
* [XEP-0004](http://wiki.jabbercn.org/XEP-0004):表单数据，用来交换数据，form类型[http://xmpp.org/registrar/formtypes.html]
* [XEP-0198](http://xmpp.org/extensions/xep-0198.html):server端的消息确认，[tigase中的配置](http://www.tigase.org/content/stream-management)，它的存在的意义在<http://op-co.de/blog/posts/XEP-0198>中已经描述的非常详细。
* XEP-199
* [XEP-0114](http://wiki.jabbercn.org/XEP-0114):Jabber组件协议

------

### 群聊室的属性

>我们设计的群已经不是标准的xmpp群了，下面的属性是对于smack或者标准的xmpp群有意义的。

* 房间名称|muc#roomconfig_roomname
* 描述|muc#roomconfig_roomdesc
* 允许占有者更改主题|muc#roomconfig_changesubject
* 最大房间占有者人数|muc#roomconfig_maxusers
* 其 Presence 是 Broadcast 的角色|muc#roomconfig_presencebroadcast
* 列出目录中的房间|muc#roomconfig_publicroom
* 房间是持久的|muc#roomconfig_persistentroom
* 房间是适度的|muc#roomconfig_moderatedroom
* 房间仅对成员开放|muc#roomconfig_membersonly
* 允许占有者邀请其他人|muc#roomconfig_allowinvites
* 需要密码才能进入房间|muc#roomconfig_passwordprotectedroom
* 密码|muc#roomconfig_roomsecret
* 能够发现占有者真实 JID 的角色|muc#roomconfig_whois
* 登录房间对话|muc#roomconfig_enablelogging
* 仅允许注册的昵称登录|x-muc#roomconfig_reservednick
* 允许使用者修改昵称|x-muc#roomconfig_canchangenick
* 允许用户注册房间|x-muc#roomconfig_registration
* 房间管理员|muc#roomconfig_roomadmins
* 房间拥有者|muc#roomconfig_roomowners

------

### 线上服务器环境配置

>tcp调优参考我之前的文章[Linux TCP参数调整](/2013/10/30/TCP参数调整)
