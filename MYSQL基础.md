# MYSQL基础

## 一、概念

#### 	1、聚集索引(聚簇索引)

​			主键即使聚集索引 一张表只能有一个聚集索引，聚集索引 B+ tree 叶子节点 存储整条记录数据 即所有叶子节点存储了整张表的数据

​			（1）如果表定义了PK，则PK就是聚集索引；

​			（2）如果表没有定义PK，则第一个not NULL unique列是聚集索引；

​			（3）否则，InnoDB会创建一个隐藏的row-id作为聚集索引；

#### 	2.普通索引

​			除主键以外的索引，叶子节点=键值+书签。Innodb存储引擎的书签就是相应行数据的主键值。

#### 	3.回表查询

​			通过遍历一个索引树，叶子节点中记录的列值，不满足查询语句中列值，还需要跟具主键值查询聚集索引，即为回表查询

#### 	4、覆盖索引

​			不需要回表查询的 即为覆盖索引

#### 	5、页

​		mysql存储 表空间-段-区-页，数据都存在页中 innodb 默认页大小16kb

举个栗子，不妨设有表：

*t(id PK, name KEY, sex, flag);*

*画外音：id是聚集索引，name是普通索引。*

表中有四条记录：

*1, shenjian, m, A*

*3, zhangsan, m, A*

*5, lisi, m, A*

*9, wangwu, f, B*



![img](https:////upload-images.jianshu.io/upload_images/4459024-8636fab05de6780b?imageMogr2/auto-orient/strip|imageView2/2/w/359/format/webp)

image

两个B+树索引分别如上图：

（1）id为PK，聚集索引，叶子节点存储行记录；

（2）name为KEY，普通索引，叶子节点存储PK值，即id；

既然从普通索引无法直接定位行记录，那**普通索引的查询过程是怎么样的呢？**

通常情况下，需要扫码两遍索引树。

例如：

*select \* from t where name='lisi';*

**是如何执行的呢？**



![img](https:////upload-images.jianshu.io/upload_images/4459024-a75e767d0198a6a4?imageMogr2/auto-orient/strip|imageView2/2/w/421/format/webp)



如**粉红色**路径，需要扫码两遍索引树：

（1）先通过普通索引定位到主键值id=5；

（2）在通过聚集索引定位到行记录；

这就是所谓的**回表查询**，先定位主键值，再定位行记录，它的性能较扫一遍索引树更低。

#### 	

## 二、索引匹配规则

#### 组合索引规则

##### 1.匹配最左原则

​	(举例：如果表中有个组合索引，where条件中包含索引中第一个字段，那么就会使用该索引)

##### 2.全值匹配  

​	where条件中包含组合索引中所有字段，则会使用该索引

##### 3.匹配列前缀 

 	like 'x%'会使用索引  like '%x%'不会使用索引

##### 4.匹配范围值 

​	例如  a>1   1<a<3

 	mysql 的范围值是40%，超出这个范围则不会命中索引。比如表里有100 行数据，40%则是 40行，如果超出这个值，则不会命 中索引，从开始到结束的位置不能超过40%。

​	使用范围条件的时候，也会使用到该处的索引，但后面的索引都不会用到

##### 5.精确匹配某一列并范围匹配另外一列

EXPLAIN SELECT * from test where  a ='1' and b>'2';
精确匹配其中一列，范围匹配另外一列。

#### 举个栗子：有联合索引 ABC

```
查询索引是否生效 explain select * from tableA where A = ‘1’

场景一：
select * from tableA where A = '1’
会命中索引，因为他是匹配最左前缀【最左匹配原则】,上述第1种生效情况

场景二：
select * from tableA where B = '1’
不会命中索引，因为按照最左匹配原则来说，最左边的列都没有，后边的就不知道是索引了

场景三：
select * from tableA where A = ‘1’ and B=‘2’ and C='3’
会命中索引，匹配的是全值匹配索引，1+2+3 这个组成了一个索引。上述第2种生效情况

场景四
select * from tableA where A like '1%'
会命中索引，匹配列前缀，上述第3种生效情况

场景五
select * from table A where A like '%1%'
不会命中索引，这是个全匹配，所以不会匹配索引。
像like查询这种，左边千万不要打%，这样效率会很低

场景六
select * from tableA where A >‘1’ and A<'5’
会命中索引，上述第4中索引生效情况。


场景七
select * from tableA where A =‘1’ and B<‘5’ and C='3’
会命中几个索引？
2个，范围查找会使用索引， A=‘1’ 使用了索引，B<‘5’ 使用了索引，而范围查找后面的则不会使用索引
如果查询中某个列使用了范围查询，则右边的列都无法使用索引查询

场景八
select * from tableA where A=‘1’ and B<'5’
会命中索引，上述第5个索引，精确匹配其中一列，范围匹配另外一列。

场景九
select * from tableA where A=‘1’ and C<'5’
会命中一个索引，则是A=‘1’ 这个索引，C<'5’不会命中索引

场景切换
如果不用联合索引，把 B 和 C 都单独拿出来做索引。
场景一
select * from tableA where A=‘1’ and B='2’
会不会命中索引，会命中几个索引
实际会命中一个索引，一个简单查询只有一个索引列生效
```

## 三、mysql阈值

1. 一个表里最多可有1017列（在MySQL 5.6.9 之前最大支持1000列）。虚拟列也受限这个限制。
2. 一个表最多可以有64个二级索引。
3. 页大小 16kb
4. 联合索引最多支持16列，如果超过这个限制就会遇到以下错误：
   ERROR 1070 (42000): Too many key parts specified; max 16 parts allowed
5. 行所存储的值的最大值为页的大小的一半，一般情况下页的大小是16kB，所以每行的长度大小不能超过8
   kb；2的16次刚好为65535；
6. 行长度（除去可变长类型：VARBINARY/VARCHAR/BLOB/TEXT），要小于页长（如4KB, 8KB, 16KB, and 32KB）的一半。例如：innodb_page_size 长度是16KB的话，行长不超过8KB；如果innodb_page_size 是64KB的话，行长不超过16KB； LONGBLOB/LONGTEXT/BLOB/TEXT列必须小于4GB，整个行长也必须小于4GB。如果一行小于一页的一半，它可以存在一个page里面。如果超过了页的一半，就会把可变长列放到额外的页存
7. 数据库维护一个连接 约需要2M内存