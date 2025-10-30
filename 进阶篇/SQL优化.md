# SQL优化-插入数据
## insert优化

* 批量插入

  ~~~sql
  Insert into tb_test values(1,'Tom'),(2,'Cat'),(3,'Jerry');
  ~~~

* 手动提交事务

  ~~~sql
  start transaction;
  insert into tb_test values(1,'Tom'),(2,'Cat'),(3,'Jerry');
  insert into tb_test values(4,'Tom'),(5,'Cat'),(6,'Jerry');
  insert into tb_test values(7,'Tom'),(8,'Cat'),(9,'Jerry');
  commit;
  ~~~

* 主键顺序插入

  ~~~
  主键乱序插入：8 1 9 21 88 2 4 15 89 5 7 3
  主键顺序插入：1 2 3 4 5 7 8 9 15 21 88 89
  ~~~

* 大批量插入数据(load)

  ~~~sql
  #客户端连接服务端时，加上参数 --local-infile。
  mysql --local-infile -u root -p
  #设置全局参数 local_infile 为 1，开启从本地加载文件导入数据的开关。
  set global local_infile = 1;
  #执行 load 指令将准备好的数据，加载到表结构中。
  load data local infile '/root/sql1.log' into table tb_user fields terminated by ',' lines terminated by '\n';
  ~~~

## 主键优化

* 页分裂
  ![image-20251027194910387](C:\Users\46814\AppData\Roaming\Typora\typora-user-images\image-20251027194910387.png)
* 页合并
  ![image-20251027195035830](C:\Users\46814\AppData\Roaming\Typora\typora-user-images\image-20251027195035830.png)
* 主键设计原则
  * 满足业务需求的情况下，尽量降低主键的长度。
  * 插入数据时，尽量选择顺序插入，选择使用AUTO_INCREMENT自增主键。
  * 尽量不要使用UUID做主键或者时其他自然主键，如身份证号。
  * 业务操作时，避免对主键的修改。

## order by优化

>①. Using filesort：通过表的索引或全表扫描，读取满足条件的数据行，然后在排序缓冲区 sort buffer 中完成排序操作，所有不是通过索引直接返回排序结果的排序都叫 FileSort 排序。
>②. Using index：通过有序索引顺序扫描直接返回有序数据，这种情况即为 using index，不需要额外排序，操作效率高。

* order by优化
  * 根据排序字段建立合适的索引，多字段排序时，页遵循最左前缀法则。
  * 尽量使用覆盖索引。
  * 多字段排序，一个升序一个降序，此时要注意联合索引创建的规则(ASD/DESC)。
  * 如果不可避免的出现 filesort，大数据量排序时，可以适当增大排序缓冲区大小 sort_buffer_size (默认 256k)

## group by优化

* group by优化
  * 在分组操作时，可以通过索引来提高效率。
  * 在分组操作时，索引的使用也满足最左前缀法则。

## limit优化

* 优化思路

  一般分页查询时，通过创建 覆盖索引 能够比较好地提高性能，可以通过覆盖索引加子查询形式进行优化。

  ~~~sql
  explain select * from tb_sku t , (select id from tb_sku order by id limit 2000000,10) a where t.id = a.id;
  ~~~

## count优化

* count的几种用法
  * **count(主键)**
    InnoDB 引擎会遍历整张表，把每一行的主键 id 值都取出来，返回给服务层。服务层拿到主键后，直接按行进行累加 (主键不可能为 null)。
  * **count(字段)**
    * 没有 not null 约束：InnoDB 引擎会遍历整张表把每一行的字段值都取出来，返回给服务层，服务层判断是否为 null，不为 null，计数累加。
    * 有 not null 约束：InnoDB 引擎会遍历整张表把每一行的字段值都取出来，返回给服务层，直接按行进行累加。
  * **count(1)**
    InnoDB 引擎遍历整张表，但不取值。服务层对于返回的每一行，放一个数字 “1” 进去，直接按行进行累加。
  * **count(*)**
    InnoDB 引擎并不会把全部字段取出来，而是专门做了优化，不取值，服务层直接按行进行累加。

>按照效率排序的话，`count(字段) < count(主键 id) < count(1) ≈ count(*) `，所以尽量使用 `count(*)`。

## update优化

* InnoDB的行级锁是针对索引加的锁，不是针对记录加的锁，并且该索引不能失效，否则会从行级锁升级为表锁。