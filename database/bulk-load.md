数据库加速load的技巧
------------------------------
1. 关闭auto commit，所有插入包成一个事务
2. 通过降低事务保证或者恢复时间来提高插入性能：a. 调整sync/wal参数 b. 延长checkpoint时间
3. 如果是导入新表的数据，可以先删除索引，插入完成后再重建索引 (如果数据不是按照索引index排列，为何能提高性能？内存能缓冲更多的索引page从而有更多机会合并减少随机IO？)
4. 删除外键约束，然后插入完成后再重建
5. pg的copy，如果不能使用copy (直接拷贝文件，如csv格式)，尽量使用prepared statements或者一条语句插入多行数据，避免sql解析的开销
6. bulk load命令特指从文件导入的命令，pg的COPY，oracle的LOAD，以及mysqlimport命令行工具

> 1. Indexes are usually optimized for inserting rows one at a time. When you are adding a great deal of data at once, inserting rows one at a time may be inefficient. For instance, with a B-Tree, the optimal way to insert a single key is very poor way of adding a bunch of data to an empty index.

> Instead you pursue a different strategy with B-Trees. You presort all of the data, and group it in blocks. You can then build a new B-Tree by transforming the blocks into tree nodes. Although both techniques have the same asymptotic performance, O(n log(n)), the bulk-load operation has much smaller factor.

> 2. Bulk loading is used to import/export large amounts of data. Usually bulk operations are not logged and transactional integrity might not work as expected. Often bulk operations bypass triggers and integrity checks like constraints. This improves performance, for large amounts of data, quite significantly.
