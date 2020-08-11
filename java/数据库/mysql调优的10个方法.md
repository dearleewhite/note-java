[MySQL 性能调优的10个方法](https://www.cnblogs.com/chenshengqun/p/8875512.html)

# [ ](http://www.cnblogs.com/claireyuancy/p/7258314.html)

MYSQL 应该是最流行了 WEB 后端数据库。WEB 开发语言近期发展非常快，PHP， Ruby, Python, Java 各有特点，尽管 NOSQL 近期越來越多的被提到，可是相信大部分架构师还是会选择 MYSQL 来做数据存储。

 

MYSQL 如此方便和稳定。以至于我们在开发 WEB 程序的时候非常少想到它。即使想到优化也是程序级别的，比方。不要写过于消耗资源的 SQL 语句。可是除此之外，在整个系统上仍然有非常多能够优化的地方。

![MYSQL 调优和使用必读](http://static.codeceo.com/images/2015/07/221658l1rrof0mfu214my4.png)

## 1. 选择合适的存储引擎: InnoDB

除非你的数据表使用来做仅仅读或者全文检索 (相信如今提到全文检索，没人会用 MYSQL 了)。你应该默认选择 InnoDB 。

你自己在測试的时候可能会发现 MyISAM 比 InnoDB 速度快。这是由于： MyISAM 仅仅缓存索引，而 InnoDB 缓存数据和索引，MyISAM 不支持事务。可是 假设你使用 innodb_flush_log_at_trx_commit = 2 能够获得接近的读取性能 (相差百倍) 。

 

#### 1.1 怎样将现有的 MyISAM 数据库转换为 InnoDB:

```
mysql -u [USER_NAME] -p -e "SHOW TABLES IN [DATABASE_NAME];" | tail -n +2 | xargs -I '{}' echo "ALTER TABLE {} ENGINE=InnoDB;" > alter_table.sql
perl -p -i -e 's/(search_[a-z_]+ ENGINE=)InnoDB//1MyISAM/g' alter_table.sql
mysql -u [USER_NAME] -p [DATABASE_NAME] < alter_table.sql
```

#### 1.2 为每一个表分别创建 InnoDB FILE：

```
innodb_file_per_table=1
```

这样能够保证 ibdata1 文件不会过大。失去控制。尤其是在运行 mysqlcheck -o –all-databases 的时候。

## 2. 保证从内存中读取数据。讲数据保存在内存中

#### 2.1 足够大的 innodb_buffer_pool_size

推荐将数据全然保存在 innodb_buffer_pool_size ，即按存储量规划 innodb_buffer_pool_size 的容量。这样你能够全然从内存中读取数据。最大限度降低磁盘操作。

 

##### 2.1.1 怎样确定 innodb_buffer_pool_size 足够大。数据是从内存读取而不是硬盘？

方法 1

```
mysql> SHOW GLOBAL STATUS LIKE 'innodb_buffer_pool_pages_%';
+----------------------------------+--------+
| Variable_name                    | Value  |
+----------------------------------+--------+
| Innodb_buffer_pool_pages_data    | 129037 |
| Innodb_buffer_pool_pages_dirty   | 362    |
| Innodb_buffer_pool_pages_flushed | 9998   |
| Innodb_buffer_pool_pages_free    | 0      |  !!!!!!!!
| Innodb_buffer_pool_pages_misc    | 2035   |
| Innodb_buffer_pool_pages_total   | 131072 |
+----------------------------------+--------+
6 rows in set (0.00 sec)
```

发现 Innodb_buffer_pool_pages_free 为 0，则说明 buffer pool 已经被用光，须要增大 innodb_buffer_pool_size

InnoDB 的其它几个參数：

```
innodb_additional_mem_pool_size = 1/200 of buffer_pool
innodb_max_dirty_pages_pct 80%
```

方法 2

或者用iostat -d -x -k 1 命令，查看硬盘的操作。

##### 2.1.2 server上是否有足够内存用来规划

运行 echo 1 > /proc/sys/vm/drop_caches 清除操作系统的文件缓存。能够看到真正的内存使用量。

#### 2.2 数据预热

默认情况，仅仅有某条数据被读取一次，才会缓存在 innodb_buffer_pool。所以，数据库刚刚启动，须要进行数据预热，将磁盘上的全部数据缓存到内存中。

数据预热能够提高读取速度。

对于 InnoDB 数据库，能够用下面方法，进行数据预热:

\1. 将下面脚本保存为 MakeSelectQueriesToLoad.sql

```
SELECT DISTINCT
    CONCAT('SELECT ',ndxcollist,' FROM ',db,'.',tb,
    ' ORDER BY ',ndxcollist,';') SelectQueryToLoadCache
    FROM
    (
        SELECT
            engine,table_schema db,table_name tb,
            index_name,GROUP_CONCAT(column_name ORDER BY seq_in_index) ndxcollist
        FROM
        (
            SELECT
                B.engine,A.table_schema,A.table_name,
                A.index_name,A.column_name,A.seq_in_index
            FROM
                information_schema.statistics A INNER JOIN
                (
                    SELECT engine,table_schema,table_name
                    FROM information_schema.tables WHERE
                    engine='InnoDB'
                ) B USING (table_schema,table_name)
            WHERE B.table_schema NOT IN ('information_schema','mysql')
            ORDER BY table_schema,table_name,index_name,seq_in_index
        ) A
        GROUP BY table_schema,table_name,index_name
    ) AA
ORDER BY db,tb
;
```

\2. 运行

```
mysql -uroot -AN < /root/MakeSelectQueriesToLoad.sql > /root/SelectQueriesToLoad.sql
```

\3. 每次重新启动数据库，或者整库备份前须要预热的时候运行：

```
mysql -uroot < /root/SelectQueriesToLoad.sql > /dev/null 2>&1
```

#### 2.3 不要让数据存到 SWAP 中

假设是专用 MYSQL server。能够禁用 SWAP，假设是共享server，确定 innodb_buffer_pool_size 足够大。或者使用固定的内存空间做缓存，使用 memlock 指令。

 

## 3. 定期优化重建数据库

mysqlcheck -o –all-databases 会让 ibdata1 不断增大。真正的优化仅仅有重建数据表结构：

```
CREATE TABLE mydb.mytablenew LIKE mydb.mytable;
INSERT INTO mydb.mytablenew SELECT * FROM mydb.mytable;
ALTER TABLE mydb.mytable RENAME mydb.mytablezap;
ALTER TABLE mydb.mytablenew RENAME mydb.mytable;
DROP TABLE mydb.mytablezap;
```

## 4. 降低磁盘写入操作

#### 4.1 使用足够大的写入缓存 innodb_log_file_size

可是须要注意假设用 1G 的 innodb_log_file_size 。假如server当机。须要 10 分钟来恢复。

 

推荐 innodb_log_file_size 设置为 0.25 * innodb_buffer_pool_size

#### 4.2 innodb_flush_log_at_trx_commit

这个选项和写磁盘操作密切相关：

innodb_flush_log_at_trx_commit = 1 则每次改动写入磁盘
innodb_flush_log_at_trx_commit = 0/2 每秒写入磁盘

假设你的应用不涉及非常高的安全性 (金融系统)，或者基础架构足够安全，或者 事务都非常小，都能够用 0 或者 2 来减少磁盘操作。

 

#### 4.3 避免双写入缓冲

```
innodb_flush_method=O_DIRECT
```

## 5. 提高磁盘读写速度

RAID0 尤其是在使用 EC2 这样的虚拟磁盘 (EBS) 的时候，使用软 RAID0 很重要。

## 6. 充分使用索引

#### 6.1 查看现有表结构和索引

```
SHOW CREATE TABLE db1.tb1/G
```

6.2 加入必要的索引

索引是提高查询速度的唯一方法。比方搜索引擎用的倒排索引是一样的原理。

索引的加入须要依据查询来确定。比方通过慢查询日志或者查询日志,或者通过 EXPLAIN 命令分析查询。

```
ADD UNIQUE INDEX
ADD INDEX
```

##### 6.2.1 比方，优化用户验证表：

加入索引

```
ALTER TABLE users ADD UNIQUE INDEX username_ndx (username);
ALTER TABLE users ADD UNIQUE INDEX username_password_ndx (username,password);
```

每次重新启动server进行数据预热

```
echo “select username,password from users;” > /var/lib/mysql/upcache.sql
```

加入启动脚本到 my.cnf

```
[mysqld]
init-file=/var/lib/mysql/upcache.sql
```

##### 6.2.2 使用自己主动加索引的框架或者自己主动拆分表结构的框架

比方。Rails 这种框架。会自己主动加入索引。Drupal 这种框架会自己主动拆分表结构。

会在你开发的初期指明正确的方向。所以，经验不太丰富的人一開始就追求从 0 開始构建，实际是不好的做法。

 

## 7. 分析查询日志和慢查询日志

记录全部查询。这在用 ORM 系统或者生成查询语句的系统非常实用。

```
log=/var/log/mysql.log
```

注意不要在生产环境用。否则会占满你的磁盘空间。

 

记录运行时间超过 1 秒的查询：

```
long_query_time=1
log-slow-queries=/var/log/mysql/log-slow-queries.log
```

## 8. 激进的方法。使用内存磁盘

如今基础设施的可靠性已经非常高了，比方 EC2 差点儿不用操心server硬件当机。并且内存实在是廉价。非常easy买到几十G内存的server，能够用内存磁盘。定期备份到磁盘。

 

将 MYSQL 文件夹迁移到 4G 的内存磁盘

```
mkdir -p /mnt/ramdisk
sudo mount -t tmpfs -o size=4000M tmpfs /mnt/ramdisk/
mv /var/lib/mysql /mnt/ramdisk/mysql
ln -s /tmp/ramdisk/mysql /var/lib/mysql
chown mysql:mysql mysql
```

## 9. 用 NOSQL 的方式使用 MYSQL

B-TREE 仍然是最高效的索引之中的一个，全部 MYSQL 仍然不会过时。

 

用 HandlerSocket 跳过 MYSQL 的 SQL 解析层。MYSQL 就真正变成了 NOSQL。

## 10. 其它

- 单条查询最后添加 LIMIT 1，停止全表扫描。
- 将非”索引”数据分离，比方将大篇文章分离存储，不影响其它自己主动查询。
- 不用 MYSQL 内置的函数。由于内置函数不会建立查询缓存。
- PHP 的建立连接速度很快，全部能够不用连接池。否则可能会造成超过连接数。当然不用连接池 PHP 程序也可能将
- 连接数占满比方用了 @ignore_user_abort(TRUE);
- 使用 IP 而不是域名做数据库路径。避免 DNS 解析问题