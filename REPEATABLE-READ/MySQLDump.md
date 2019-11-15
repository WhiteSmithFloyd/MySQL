# MySQLDump

`MySQL Dump`经常用于迁移数据和备份.

下面创建实验数据,两个数据库和若干表

```sql
create database db1 ;
use db1;
create table t1(id int primary key);
insert into t1 values(1),(2),(3);
create table t2(id int primary key);
insert into t2 values(1),(2),(3);
create database db2;

use db2;
create table t3(id int primary key);
insert into t3 values(1),(2),(3);
create table t4(id int primary key);
insert into t4 values(1),(2),(3);

commit;
```

mysqldump的常用参数如下
1.导出指定数据库(--databases)
`mysqldump -uroot --databases db1 db2 > test.sql`

2.导出指定数据库的结构（-d）
`mysqldump -uroot --databases -d db1 db2 > test.sql`

3.导出之前刷新日志(-F)

4.设置字符集(`--default-character-set`)

5.设置扩展`Insert`(`-e --skip-extended-insert`禁用扩展`Insert`)

6.锁表(`--lock-tables`)

7.锁所有数据库的所有表(`--lock-all-tables`)

8.一致性读,仅针对`InnoDB`有效（`--single-transaction`）

9.获取`binlog`位置(`--master-data` 1将`binlog`位置写在正文 2将`binlog`位置写入注释)

常用用法
1.迁移数据
将db1 db2数据库通过mysqldump导出.然后通过管道导入目标数据库
`mysqldump -uroot --single-transaction--databases db1 db2 | mysql -uroot -p123456 -h 172.16.1.25`

2.导出数据备份或者创建Slave
`mysqldump -uroot --single-transaction --master-data --databases db1 db2 > test.sql`

3.分别产生表结构和数据
`select into outfile`是针对单个表的.使用--tab选项可以导出多个表
`mysqldump -uroot --single-transaction  --tab=F:\  db1`


重要参数解析(MySQL 5.6.14)
开启MySQL `general_log`, 然后使用`mysqldump`操作,查看产生的日志.

1. `--lock-tables`

执行命令
`mysqldump -uroot --lock-tables --databases db1 db2 > test.sql`
它在导出db1的时候,会对db1所有的表上锁,导出结束之后释放锁.然后再同样导出db2.
也就是说在db1导出的时候,db2的数据可能还在变化.
![img](http://img.blog.itpub.net/blog/attachment/201501/7/29254281_1420563710TeQw.png?x-oss-process=style/bb)

2.`--lock-all-tables`
`mysqldump -uroot --lock-all-tables --databases db1 db2 > test.sql`
它会在一开始就对所有的数据库的所有表上锁, 

**请注意它会使用FLUSH TABLES**
![img](http://img.blog.itpub.net/blog/attachment/201501/7/29254281_1420564325CW7c.png?x-oss-process=style/bb)

3.`--single-transaction`
`mysqldump -uroot --single-transaction --databases db1 db2 > test.sql`
可以看到它设置整个导出的过程为一个事务.避免了锁
![img](http://img.blog.itpub.net/blog/attachment/201501/7/29254281_1420564802NuUG.png?x-oss-process=style/bb)

4.`--master-data`
它对所有数据库的所有表上了锁,并且查询binlog的位置。**请注意它会使用FLUSH TABLES**
![img](http://img.blog.itpub.net/blog/attachment/201501/7/29254281_1420565215GLL8.png?x-oss-process=style/bb)

5.`--master-data + --single-transaction`
`mysqldump -uroot --master-data --single-transaction --databases db1 db2 > test.sql`
这种组合,会先对所有数据库的所有表上锁,读取binlog的信息之后就立即释放锁,这个过程是十分短暂的。
然后整个导出过程都在一个事务里.
**请注意它会使用FLUSH TABLES**
![img](http://img.blog.itpub.net/blog/attachment/201501/7/29254281_1420566953yd22.png?x-oss-process=style/bb)

**MySQLDump在使用之前一定要想到的事情** 

如果`mysqldump`执行的过程中需要`flush tables`,而正在此时,有一个慢SQL正在运行,这时`mysqldump`会被阻塞（`waiting for table flush`）,
并且其他连接对这个表的所有操作（甚至查询）都被阻塞.系统Hung了.

这个问题在`XtraBackup`备份的时候同样存在.

如果是人工执行,一定要开启另外一个连接,监控 `show processlist`，查看是否阻塞.
如果是调度执行,拼人品了.

其实优化慢SQL才是正道.

另外在`mysqldump`导出的过程中,不要有任何的`DDL`操作,否则同样会引发`metadata lock`的连环阻塞.
