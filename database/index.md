database index
==============

索引的作用
-------------
数据库索引主要有两个作用：
1. 空间换时间，加快查询速度
2. 实现关系数据库约束(```UNIQUE```, ```EXCLUSION```, ```PRIMARY KEY``` and ```FOREIGN KEY```)

> Some database systems support EXCLUSION constraint which ensures that for a newly inserted or updated record a certain predicate would hold for no other record. This may be used to implement a UNIQUE constraint (with equality predicate) or more complex constraints, like ensuring that no overlapping time ranges or no intersecting geometry objects would be stored in the table. An index supporting fast searching for records satisfying the predicate is required to police such a constraint. [from wikipedia](http://en.wikipedia.org/wiki/Database_index)

clustered index v.s. non-clustered index
-------------

数据库索引有聚集索引和非聚集索引之分。
> 聚集索引是指记录**物理存储**顺序跟索引顺序**基本**一致，利于批量操作(scan, sort, agg等)
> 非聚集索引则无此限制，所以一个表最多只能有一个聚集索引和多个非聚集索引，注意聚集索引跟主键(_的_)索引没有必然联系

> b-tree：所有结点都存有数据
> b+ tree: 只有叶子结点存储数据

Case study:
  1. InnoDB: 索引和数据存在一棵b+树上，所以不宜采用b-tree。 [related posts](http://blog.jcole.us/2013/01/10/btree-index-structures-in-innodb/)
  2. PostgreSQL: 索引和数据分开存储，采用b-tree (应该比b+tree好些)，但是pg并不默认选取cluster index，但是可以手动通过CLUSTER命令指定任何索引，对heap file根据指定的索引按顺序重组。[links](http://stackoverflow.com/questions/4796548/about-clustered-index-in-postgres)

实际上pg和innodb区别不大，索引部分跟数据是不在同一块的，都可以保证数据的聚集性(pg 手动执行)

除了两种索引之外，Oracle还支持Clustered Table，用于多个经常会一起join的表按照join的key在物理上存储在一起。
> When multiple databases and multiple tables are joined, it's referred to as a cluster (not to be confused with clustered index described above). The records for the tables sharing the value of a cluster key shall be stored together in the same or nearby data blocks. This may improve the joins of these tables on the cluster key, since the matching records are stored together and less I/O is required to locate them.[2] The data layout in the tables which are parts of the cluster is defined by the cluster configuration. A cluster can be keyed with a B-Tree index or a hash table. The data block in which the table record will be stored is defined by the value of the cluster key. (from wikipedia)

> Clusters are an optional method of storing table data. A cluster is a group of tables that share the same data blocks because they share common columns and are often used together. For example, the employees and departments table share the department_id column. When you cluster the employees and departments tables, Oracle physically stores all rows for each department from both the employees and departments tables in the same data blocks. (from Oracle manual)


其他分类
-----------------------
1. bitmap index: (e.g. 性别)
2. dense index: 每个记录有一个索引
3. sparse index: 每个物理块有一个索引
4. inverse index: 对于顺序增长的索引列反序后做索引可以均衡负载 (如HBase PK为时间戳时)

Case study:
* leveldb: 全局缓存打开的sstable的FileMeta结构，里面保存了index BlockHandle，对应的块中保存指向数据块的稀疏索引，每个块中保存若干restarts offset，类似与索引作用，用于二分查找

  ```
  Q: How does TukoDB orgnize it's index?
  ```
* Cosmos Structured Streams: 可以是 `CLUSTERED BY` or `HASH CLUSTERED BY`，用于优化Reducer/Join等操作:
  ```
    OUTPUT TO SSTREAM “MySStream.ss” CLUSTERED BY Column1;
  ```
