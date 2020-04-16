## 前言

在很多业务场景中，我们会出现如下的需求：在某一个表中查询“热”数据，查询不到再去另一个表中查找“冷”数据，此时我们如何通过sql语句实现呢？

## 正文

* 首先，创建student和student_2两个表，如下：

  ```sql
  CREATE TABLE `student` (
    `id` bigint(20) NOT NULL,
    `name` varchar(100) DEFAULT NULL,
    PRIMARY KEY (`id`)
  ) ENGINE=InnoDB
  ```

  ```sql
  CREATE TABLE `student_2` (
    `id` bigint(20) NOT NULL,
    `name` varchar(100) DEFAULT NULL,
    PRIMARY KEY (`id`)
  ) ENGINE=InnoDB
  ```

* 在student表和student_2表分别插入数据，如下：

  ```sql
  +----+------+
  | id | name |
  +----+------+
  |  1 | haha |
  |  2 | hehe |
  +----+------+
  ```

  ```sql
  +----+------+
  | id | name |
  +----+------+
  |  2 | wewe |
  |  3 | wowo |
  +----+------+
  ```

* 前情完成，开始测试，首先在student和student_2中查询id为2的数据，如下：

  ```sql
  select * from student where id=2 union all select * from student_2 where id=2;
  ```

  ```sql
  +----+------+
  | id | name |
  +----+------+
  |  2 | hehe |
  |  2 | wewe |
  +----+------+
  ```

  可以发现，即使在student表中已经找到了记录，上面的sql还是会再去student_2表中查询id=2为数据；

* 此时，引入一个自定义变量@found，我们通过GREATEST函数和found变量来实现以上的需求

  * GREATEST函数：求的是某几列中的最大值，横向求最大(一行记录)，如greatest (a,b,c,d,d)

  ```sql
  select greatest(@found:=-1,id) as id,name from student where id=2 
  union all select id,name from student_2 where id=2 and @found is null;
  ```

  发现以上sql是可以实现功能的

  ```sql
  +----+------+
  | id | name |
  +----+------+
  |  2 | hehe |
  +----+------+
  ```

  但是，当使用该sql查询id为3的命令时，发现empty set，不合理啊？student表中找不到，应该要会返回student_2表中的数据啊，此时我们查看found的值，如下：

  ```sql
  +--------+
  | @found |
  +--------+
  |     -1 |
  +--------+
  ```

  发现found设置为-1后需要还原置null，否则影响下一步sql；

* 修改以上sql如下：

  ```sql
  select greatest(@found:=-1,id) as id,name from student where id=3 
  union all select id,name from student_2 where id=3 and @found is null 
  union all select 1,'reset' from dual where (@found:=NULL) is not null;
  ```

  解释下：

  * DUAL：为一个mysql提供进行简单操作的表，如，可以进行四则运算：select 100*199 from dual，我们再次通过dual，判断当found不为Null时来重设found的值为null；

  ```sql
  +----+------+
  | id | name |
  +----+------+
  |  3 | wowo |
  +----+------+
  ```

  验证成功，大功告成！

## 总结

假期放纵了，已许久没更博。

此次在看《高性能Mysql》中，看到此之前没涉猎到的知识点，感觉挺有收获的，故在此分享下！

闲时多读书，拒绝负能量。武汉加油，中国加油！

