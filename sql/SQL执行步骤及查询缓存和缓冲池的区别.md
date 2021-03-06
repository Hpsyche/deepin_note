## 当我们向MySQL发送一个请求的时候，MySQL到底做了什么：

* 客户端发送一条查询给服务器
* 服务器先检查**查询缓存**,如果命中了缓存,则立刻返回存储在缓存中的结果。否则进入下一阶段。
* 服务器端进行SQL解析、预处理,再由优化器生成对应的执行计划。
* MySQL根据优化器生成的执行计划,调用存储引擎的API来执行查询
* 将结果返回给客户端。

（1） MySQL客户端/服务器通信协议

MySQL客户端和服务器之的通信协议是“半双工”的，这就意味着，在任何一个时刻,要么是由服务器向客户端发送数据,要么是由客户端向服务器发送数据,这两个动作不能同时发生。所以我们无法也无须将一个消息切成小块独立来发送。

优缺点：这种协议让MySQL通信简单快速,但是也从很多地方限制了 MySQL。一个明显的限制是,这意味着没法进行流量控制。一旦一端开始发送消息,另一端要接收完整个消息才能响应它。这就像采回抛球的游戏:在任何时刻,只有一个人能控制球,而且只有控制球的人才能将球抛回去(发送消息)。

（2）查询缓存在解析一个查询语句之前,如果查询缓存是打开的，MySQL会检查这个缓存，是否命中查询缓存中的数据。这个检查是通过一个大小写敏感的哈希查找实现的。查询和缓存中的查询即使只有一个字节不同,那也不会匹配缓存结果,这种情况下查询就会进入下一阶段的处理。

如果当前的查询恰好命中了查询缓存,那么在返回查询结果之前 MySQL会检查一次用户权限。这仍然是无须解析查询SQL语句的,因为在查询缓存中已经存放了当前查询需要访问的表信息。如果权限没有问题, MySQL会跳过所有其他阶段,直接从缓存中拿到结果并返回给客户端。这种情况下,查询不会被解析,不用生成执行计划,不会被执行.

ps:**注意在 mysql8 后已经没有查询缓存这个功能了，因为这个缓存非常容易被清空掉，命中率比较低。**

(3)分析器

既然没有查到缓存，就需要开始执行 sql 语句了，在执行之前肯定需要先对 sql 语句进行解析。

分析器主要对 sql 语句进行**语法和语义分析，检查单词是否拼写错误，还有检查要查询的表或字段是否存在。**

(4)查询优化

查询的生命周期的下一步是将一个SQL转换成一个执行计划, MySQL再依照这个执行计划和存储引擎进行交互。这包括多个子阶段:**解析SQ√L、预处理、优化SQ执行计划。**这个过程中任何错误(例如语法错误)都可能终止查询。

（5）执行器

经过上面流程，接下来就开始真正的执行MySQL语句了；

- 第一步先判断是否具有对这个表执行查询的权限，如果没有，就会返回没有权限的错误；
- 如果拥有权限，就打开这个表，**接下来执行器会根据表的引擎定义，去使用这个引擎提供的接口；**

## 查询缓存与缓冲池

### 关于查询缓存

判断缓存命中的方法很简单：缓存存放在一个引用表中，通过一个哈希值引用。

查询缓存保存查询返回的完整结果。当查询命中该缓存, MySQL会立刻返回结果跳过了 解析，优化和执行阶段

查询缓存系统会跟踪查迫中涉及的每个表,如果这些表发生变化,那么和这个表相关的的存数据都将失效。

这种机制效率看起来比较低,因为数据表变化时很有可能对查询结果并没有变更,但是这种简单实现代价很小,而这点对于一个非常繁忙的系统来说非常重要。

查询缓存系统对应用程序是完全透明的。应用程序无须关心 MySQL是通过查询缓存返回的结果还是实际执行返回的结果。事实上,这两种方式执行的结果是完全相同的。换句话说,查询缓存无须使用任何语法。无论是 MYSQL开启成关闭查询缓在,对应用程序都是透明的。

query_cache_type

是否打开查询缓存。可以设置成0FN或 DEMAND。 DEMAND表示只有在查询语句中明确写明SQL_ CACHE的语句才放入查询缓存。这个变量可以是会话级别的也可以是全局级别的

query_cache_size

查询缓存使用的总内存空间,单位是字节。这个值必须是1024的整数倍,否则 MySQL实际分配的数据会和你指定的略有不同。

### 关于缓冲池

缓冲池概念：缓冲池简单来说就是一块内存区域，通过内存的速度来弥补磁盘速度较慢对数据库性能的影响。**在数据库当中读取页的操作，首先将从磁盘读到的页存放在缓存池中，这个过程称为将页“FIX”在缓冲池中。下一次再读相同的页时，首先判断该页是不是在缓冲池中。若在，直接读取。否则，读取磁盘上的页。**

那么如果sql语句修改了缓存池的页的数据，数据是怎么同步到磁盘保存的？

对于数据库中页的修改操作，则首先修改缓存池中的页，然后再以一定的频率刷新到磁盘上。注意：缓冲池刷新回磁盘并不是每次页发生更新时触发，而是通过一种称为Checkpoint的机制刷新回磁盘。这样，是为了进一步提高数据库整体性能。

### 缓冲池与查询缓存的区别

缓冲区：

* innodb大小：(innodb_buffer_pool_size)默认为256MB，比查询缓存大，建议为服务器内存的75%-80%

* myisam大小：（key_buffer_size）默认为132MB

  * Myisam引擎有Key Cache：专门缓存索引，淘汰算法LRU 
    Innodb引擎有buffer pool：缓存数据和索引，淘汰算法LRU

  * 为什么MyISAM会比Innodb 的查询速度快。

    INNODB在做SELECT的时候，要维护的东西比MYISAM引擎多很多；
    1）数据块，INNODB要缓存，MYISAM只缓存索引块， 这中间还有换进换出的减少；
    2）innodb寻址要映射到块，再到行，MYISAM 记录的直接是文件的OFFSET，定位比INNODB要快
    3）INNODB还需要维护MVCC一致；虽然你的场景没有，但他还是需要去检查和维护

* 作用于存储引擎层

查询缓存：

* 大小（query_cache_size），默认为64MB
* 作用于服务器层

