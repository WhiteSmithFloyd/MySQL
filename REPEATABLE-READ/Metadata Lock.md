# Metadata Lock

他的本意是解决之前版本事务隔离特性的几个bug,但是引入的问题也不小.

先说说MySQL的事务吧.
Oracle的事务指的是需要分配回滚段的SQL语句,也就是说select并不是oracle事务的一部分.
比如运行一个查询,然后在另外一个会话查询v$transaction,并不会有任何相关的信息.直到事务中出现insert,update,delete。
而innodb的事务包括select查询.
无论事务隔离级别是可重复读,还是读提交,只要有查询,事务就开始了
下图证明了在5.6.15,设置了autocommit=0之后,运行一个查询就可以开启一个事务.
第一个会话运行查询.
![img](http://img.blog.itpub.net/blog/attachment/201412/28/29254281_1419776656mWxE.png?x-oss-process=style/bb)

第二个会话,运行 show engine innodb status\G 查看事务情况
![img](http://img.blog.itpub.net/blog/attachment/201412/28/29254281_1419776895j4Fd.png?x-oss-process=style/bb)
可以看到id为1的线程,已经开始了一个事务.

为什么Oracle的事务仅包括insert,update和delete的语句,而innodb的事务包括所有的语句呢?
我觉得这个和厂商支持的隔离级别有很大的关系.
众所周知,Oracle仅仅支持读提交和串行化两种事务隔离级别,而读提交是绝大多数数据库的选择.
读提交意味着可以出现幻读和不可重复读,那么从实现原理的角度,Oracle可以在语句(Statement级别)开始的时候,记录SCN然后应用MVCC查询.每个查询只需要记录自己开始的SCN即可.而语句开始的SCN和事务并没有关系.所以Oracle的事务,并不包括查询.

而innodb支持可重复读隔离级别,也就是说在一个事务中,无论运行多少次查询,结果都必须是一致的.
(innodb不仅支持可重复读,并且使用间隙锁在可重复读级别避免了幻读,当然这也带来了很多问题..)
所以它记录的不是每个查询语句的LSN,而是事务第一个语句发生时的LSN,无论第一个语句是查询,还是修改.
innodb在可重复读的级别下,查询用事务开始时的LSN应用MVCC,与Oracle不同的是,innodb查询回滚段中小于事务开始的LSN的数据版本,
而oracle查询回滚段中小于语句SCN的数据版本.
也就是说,同样都是MVCC,oracle是语句级的,innodb是事务级的

这里有一个问题,按说事务包括查询是因为可重复读隔离级别的需要,但是innodb读提交隔离级别同样也将查询作为了事务的一部分.
可能是因为架构或者代码实现层面的问题吧.
不管怎么样,Innodb就是这么做了.

然后再说说metadata lock
在5.5.3之前,metadata lock是语句级的,这实际上破坏了事务的一致性.
比如一个事务,在可重复读隔离级别,运行两次查询,居然结果不一致.
![img](http://img.blog.itpub.net/blog/attachment/201412/28/29254281_1419778825aJ77.png?x-oss-process=style/bb)
这正是因为metadata lock是语句级造成的问题,
在两个查询的间隔,另外一个会话执行了truncate table.
所以再次运行查询,没有任何结果.

MySQL为了解决这个问题,在5.5.3将metadata lock提升为事务级别的锁.
任何DDL都需要先获得metadata lock,但是这个锁需要等事务结束的时候释放.
同样的实验,在5.6.13就变成这样的了.
第一个会话的事务没有结束,那么第二个会话的DDL就被阻塞
![img](http://img.blog.itpub.net/blog/attachment/201412/28/29254281_1419779274kyjy.png?x-oss-process=style/bb)
使用show processlist可以看到DDL语句在等待第一个会话事务的metadata lock
![img](http://img.blog.itpub.net/blog/attachment/201412/28/29254281_1419779369ciJI.png?x-oss-process=style/bb)
通过这种方式,就保证了可重复读隔离级别下,事务的一致性.

和之前提到的查询也作为事务的一部分一样,innodb并没有为读提交量身定制一些东西,
比如读提交并不需要查询作为事务的一部分
和读提交并不需要事务级别的metadata lock.
可能是出于架构层面的问题,很多可重复读的特性强加在了读提交上,
所以一旦这些特性出现问题,即使将隔离级别降为读提交也不能避免.

接下来问题来了,
刚才的DDL被metadata lock阻塞,这个DDL还会进一步阻塞其他的事务.甚至是查询(查询是innodb事务的一部分.)
![img](http://img.blog.itpub.net/blog/attachment/201412/28/29254281_1419780212s1X8.png?x-oss-process=style/bb)
这就有点抓狂了,因为这个时候,系统其实已经Hung了.
假设id为1的线程持有metadata lock 没有提交,
id为2的线程进行DDL,然后被阻塞在线程1的metadata锁上,
这时,数据库依次来了8个查询,他们都阻塞在了线程2上.
假如线程1的事务不结束,其他的线程都被阻塞.
即使线程1的事务结束了..也是后面8个事务依次获得metadata锁,与此同时,这个DDL可能又阻塞了80个事务..

这时候,系统的并发为1,这个DDL可能永远不能执行.并且这种情况不在死锁检测的范围内.
它的锁超时时间,由lock_wait_timeout参数控制,默认是31536000(一年,坑爹吧)

MySQL虽然保证了事务的一致性,避免了bug,但是引入的问题却可能让我这样的初级dba丢了饭碗..

最后梳理一下可能引发metadata lock连环阻塞的情况
1.在有其他事务运行的时候,进行DDL操作(alter table;truncate;)
2.在mysqldump运行的时候,进行DDL操作.(想想就觉得坑爹)
3.在Master-Slave复制环境,在Slave运行查询,会导致Master传过来的DDL阻塞.导致复制延迟增大.
4.创建索引(...)

作为初级dba来说,为了保住饭碗,可以有两个动作
1.将lock_wait_timeout参数调低
2.在运行DDL之前,查看事务是否频繁,在运行DDL之后,开启另外一个会话,使用show processlist查看是否被metadata lock阻塞.
一旦阻塞,先Kill ddl的操作.
