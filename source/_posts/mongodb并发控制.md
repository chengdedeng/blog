title: MongoDB并发控制
date: 2012-5-19 11:02:29
categories: 技术
tags: [Nosql,数据库]
------
>MongoDB在我们的生产环境中已经大规模的使用，它的性能与稳定已经得到的充分的验证，稳定在线的时间已经有一年多了。在这个过程中的确给我们带来了很多性能上的优势，虽然它不像关系型数据那样有方便的join查询，但就目前我们的应用场景这些缺点（暂且把它当做缺点吧）都是可以接受的。最近在思考了下nosql数据库并发控制方面的问题，在此记录一下。



数据库的**并发控制**机制不外乎就两种情况：


1. [悲观锁](http://zh.wikipedia.org/wiki/%E6%82%B2%E8%A7%82%E5%B9%B6%E5%8F%91%E6%8E%A7%E5%88%B6)：假定会发生并发冲突，屏蔽一切可能违反数据完整性的操作。
2. [乐观锁](http://zh.wikipedia.org/wiki/%E4%B9%90%E8%A7%82%E5%B9%B6%E5%8F%91%E6%8E%A7%E5%88%B6)：假设不会发生并发冲突，只在提交操作时检查是否违反数据完整性，乐观锁不能解决脏读和多读的问题。
悲观锁假定其他用户企图访问或者改变你正在访问、更改的对象的概率是很高的，因此在悲观锁的环境中，在你开始改变此对象之前就将该对象锁住，并且直到你提交了所作的更改之后才释放锁。悲观的缺陷是不论是页锁还是行锁，加锁的时间可能会很长，这样可能会长时间的限制其他用户的访问，也就是说悲观锁的并发访问性不好。
乐观锁则认为其他用户企图改变你正在更改的对象的概率是很小的，因此乐观锁直到你准备提交所作的更改时才将对象锁住，当你读取以及改变该对象时并不加锁。可见乐观锁加锁的时间要比悲观锁短，乐观锁可以用较大的锁粒度获得较好的并发访问性能。但是如果第二个用户恰好在第一个用户提交更改之前读取了该对象，那么当他完成了自己的更改进行提交时，数据库就会发现该对象已经变化了，这样，第二个用户不得不重新读取该对象并作出更改。这说明在乐观锁环境中，会增加并发用户读取对象的次数。

<!--more-->

MongoDB不支持事务，也就没有数据库级别的悲观锁了。那么我们的并发控制只能依赖乐观锁，乐观锁非常适合我的应用场景，并且性能更高。刚才说的是数据库级别的并发控制，当然如果说到程序级别并发控制机制，同样是悲观锁和乐观锁。我们的经常用的lock就是一种悲观锁，不论是`java`还是`.net`。那乐观锁呢？[软件事务内存](http://zh.wikipedia.org/wiki/%E8%BD%AF%E4%BB%B6%E4%BA%8B%E5%8A%A1%E5%86%85%E5%AD%98)，[Clojure](http://zh.wikipedia.org/wiki/Clojure)和[Scala](http://zh.wikipedia.org/wiki/Scala)言语的内存管理都大量使用了乐观锁，因此它们天生就是支持并发编程的。

--------------

如果要在普通的关系型数据库里实现乐观并发控制，我们一般需要为其加上一个额外的Version字段，它是整型，也可能是个时间戳。在更新某条记录时，我们将这个字段的“旧值”作为UPDATE语句的条件之一，同时这个字段也会写入新的值。如果这次更新影响了某条记录，那么表示更新成功，反之则表示这条记录已经被删除，或是在“读取”和“提交”之间遇到了其他提交操作。在SQL Server中存在一个Timestamp类型，这个类型的字段会在记录修改时自动更新。


要在MongoDB实现乐观锁，方式差不多，只是update完了之后，不会返回修改的数据条数，得还要自己去查询一下是否修改成功。

``` bash
>db.log.update({"uuid":"1",version:1},{$set:{"enddate":"2012-5-19 17:17:10",version:2}})
> db.$cmd.findOne({getlasterror:1}) {   "updatedExisting" : true,     "n" : 1,        "connectionId" : 1, "err" : null,        "ok" : 1 }
>db.log.update({"uuid":"1",version:1},{$set:{"enddate":"2012-5-19 17:17:10",version:2}})
> db.$cmd.findOne({getlasterror:1}) {   "updatedExisting" : false,     "n" : 0,        "connectionId" : 1, "err" : null,        "ok" : 1 }   
```


在update语句后面跟上一句db.$cmd查询，如果它返回updatedExisting为true，则表示更新成功了。当然如果使用java驱动的话，可以使用dbCollection.update和dbCollection.getStats()，便可以更新并且返回状态信息。但是db.$cmd查询的结果是否准确呢？如果在update语句和db.$cmd查询之间，如果另外一个连接恰好也执行了一次update操作，那么db.$cmd返回的是哪次更新的结果？通过查询官方资料，db.$cmd查询是与连接相关，这便不会有问题了。不过值得注意的是，驱动程序是“自动管理连接”的，也就是说当update()完成之后getstats()有可能使用的不是同一个链接了，这个时候db.$cmd返回的状态信息就不准确。所以如果采用上诉方式你要确保自己两次获得的是同一个链接。如果你想一直使用同一个连接，可以用下边这种方式：

``` bash
DB db...;
db.requestStart();
code....
db.requestDone();
```
但是如果最后db.requestDone()没有被调用，该连接不会被交还给线程池，所以，一定要在finally块中调用db.requestDone()。

由于我使用spring来管理mongo驱动，我不喜欢上面那种保持一个链接的方式，所以我用[findAndModify](http://docs.mongodb.org/manual/reference/command/findAndModify/)来更新数据，`当然更新条件自然同样需要包括版本号。如果更新成功，那么findAndModify命令则会返回“更新前”的数据，否则则返回空文档`。这种使用数据库命令行的方式就可以避免保持同一个链接的要求。该方式也是我推荐的方式，官方文档已经说的很清楚了。
>This command can be used to atomically modify a document (at most one) and return it. Note that, by default, the document returned will not include the modifications made on the update.

以上便是mongodb并发控制乐观锁的实现方式。
