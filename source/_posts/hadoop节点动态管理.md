title: Hadoop节点动态管理
date: 2012-11-30 23:42:29
categories: 技术
tags: [bigData]
------
>hadoop在线运行已经很长一段时间了，下面就是我在上下线datanode和tasktracker步骤。因为datanode节点不一定是tasktracker，即使datanode和tasktracker在同一节点，你也可能只上下线其中一个，所以我在配置dfs和mr的include和exclude的时候，分开配置。

### namenode中hdfs-site.xml配置

``` xml
	 <property>
		 <name>dfs.hosts</name>
		 <value>/ddmap/hadoop-1.0.4/conf/hdfs_include</value>
	 </property>
	 <property>
		 <name>dfs.hosts.exclude</name>
		 <value>/ddmap/hadoop-1.0.4/conf/hdfs_exclude</value>
	 </property>
```
dfs.hosts所对应的文件中列出了所有可以连接到namenode的datanode，如果为空则所有的都可以连入。dfs.hosts.exclude所对应的文件中列出了禁止连接namenode的datanode节点。如果一个节点在这两个文件中都存在，则不允许连入。

<!--more-->

------

### 下线datanode步骤

1. 将要下线的机器加入到dfs.hosts.exclude指定的文件中（使用主机名，ip基本不靠谱），然后刷新配置hadoop dfsadmin -refreshNodes。
2. 通过hadoop dfsadmin -report或者web界面，可以看到，该datanode状态转为Decommission In Progress。
3. 当decommission进程完成数据移动，datanode状态会转变为Decommissioned，然后datanode会自动停止datnode进程。然后你可以看见dead nodes下多了一个你想要下线的节点。
4. 然后删除include和exclude中该节点的hosts，重新刷新hadoop dfsadmin -refreshNodes。
5. 最后别忘了删除slaves中该节点的配置，防止下次整个集群重启时，该节点不能通过namenode自动启动。

**注意**:当你下线一个datanode节点，有可能该节点长时间处于Decommission In Progress状态，一直不能转变为Decommissioned。请你用hadoop fsck /检查下是否有些块少于指定的块数，特别注意那些mapreduce的临时文件。将这些删除，并且从垃圾箱移除，该节点就可以顺利下线。这是我解决该问题的办法。

----
### 上线一个节点的步骤
1. 保证将要上线的机器不存在于dfs.hosts.exclude所对应的的文件中，并且存在于dfs.hosts所对应的文件中。
2. 在namenode上刷新配置：hadoop dfsadmin -refreshNodes。
3. 在要上线的节点重启datanode,hadoop-daemon.sh start datanode。
4. 通过hadoop dfsadmin -report或者web界面，可以看到，节点已经上线。
5. 还是老话最后别忘了修改slaves。

----
### jobtracker中mapred-site.xml的配置

``` xml
	 <property>
		 <name>mapred.hosts</name>
		 <value>/ddmap/hadoop-1.0.4/conf/mr_include</value>
	 </property>
	 <property>
		 <name>mapred.hosts.exclude</name>
		 <value>/ddmap/hadoop-1.0.4/conf/mr_exclude</value>
	 </property>
```

tasktracker上下线基本同上。mapred.hosts中的文件是允许连接到jobtracker的机器，mapred.hosts.exclude则刚好相反，如果一台机器在两个配置中都存在，则不允许连接到jobtracker。

---------
### 下线tasktracker步骤
1. 将要下线的机器加入到mapred.hosts.exclude指定的文件中（使用主机名，ip基本不靠谱）。
2. 刷新配置：hadoop mradmin -refreshNodes，你可以看见web界面Excluded Nodes中多出了一个节点，该节点就是你要下线的节点，最后到你要下线的节点上运行hadoop-daemon.sh stop tasktracker。
3. 最后将mr_exclude和mr_include中该节点的hosts删除，并在jobtracker上再次刷新该配置，hadoop mradmin -refreshNodes。
4. 再次提醒别忘了修改slaves。

tasktracker上线基本上和datnode上线原理和步骤都差不多，此处就不说了。
tasktracker上下线问题不是很大，最多就是任务失败，我想这个对于很多人来说都是可以忍受的，但是datanode上下线需要注意，因为datanode上下线如果不得当，可能导致数据丢失，特别是同事操作2个以上的节点。如果集群不是非常大，最好的每次下线一至两台，通过检查发现数据没有问题的时候再下线别的。这样虽然麻烦，但是是最安全的。
