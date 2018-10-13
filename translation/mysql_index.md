# MySQL5.6 索引最佳实践

>
这是 [文章](https://www.percona.com/live/europe-amsterdam-2015/sites/default/files/slides/PLAM15-MySQL-Indexing-Best-Practices-for-MySQL-56.pdf) 的翻译，在翻译过程中，会对其中涉及到的语句加上一些个人理解以及 SQL 语句的执行，并进行特别的标注。

### 1. 你做了一个很棒的选择，因为：
* 对于普通开发者和 DBA，理解索引都是非常重要的；
* 对于大量的生产环境上的问题，糟糕的索引要负有责任；
* 索引没有非常的高深。

### 2. MySQL 索引事项
* 理解索引；
* 为自己的应用选择最好的索引；
* 解决常见的 MySQL 限制。

### 3. 废话少数，索引有什么用？
* 可以更快的访问数据库；
* 可以增加强制限制（UNIQUE， FOREIGN KEY）；
* 没有索引的查询可以运行，但是可能会花费很长时间。

### 4. 你可能听过的一些索引类型
* B-Tree 索引：MySQL 中最主要的索引；
* RTREE 索引：仅仅是 MyISAM，GIS；
* 哈希索引：MyISAM，5.6 开始的 Innodb。

### 5. BTREE 索引家族
* 多种不同的实现：
   - 为了加速，都有相同的操作；
   - 内存和磁盘是需要考虑的两方面；

* B+Tree 是典型的磁盘存储：数据存储在页节点上；
* TokuDB Fractal Trees 在逻辑上是相似的：但是在物理存储上不同

### 6. B+ 树例子
![图片](../static/btree_example.jpg)

### 7. MyISAM vs Innodb
* 在 MyISAM 中，数据指针指向数据文件的物理节点，所有的索引都是等价的；
* 在 Innodb 中，主键在页节点上存储数据，耳机索引存储主键作为数据指针。

### 8. BTREE 索引能做什么？
* 直接查看 KEY=5 的所有列；
* 找到 KEY > 5 的列，范围查找；
* 查找 5<KEY<10 之间的所有列，封闭范围查找；
* 不能找到 KEY 的最后一个数字是 0 的列（这个不是范围查找）。

### 9. 字符串索引
* 字符串索引实际上也没什么不同，按照字典顺序排列，例如 ”AAAA" < "AAAB";
* like 前缀是一个特殊排序，例如 LIKE “ABC%”  意味着 “ABC[LOWEST]”<KEY<“ABC[HIGHEST]”，但是 LIKE “%ABC” 不走索引。

### 10. 多列索引
* 按照定义的顺序从左往右进行比较，例如在 KEY(col1,col2,col3) 中，(1,2,3) < (1,3,1)；
* 多列索引仍然是一个 BTREE 索引，但不是每列都是一个单独的 BTREE。

### 11. 索引注意事项：
* 索引会有消耗，不要加比需要更多的索引，在多数情况下拓展现有的索引比加新索引更好；
* 更新索引会造成数据库大量的写操作；
* 索引会在磁盘和内存上浪费空间，但是会优化查询性能。

### 12. Innodb 索引
* 数据通过主键聚集存储：
	- 通过主键查询是最好的方式；
	- 对于文章的评论，(POST_ID,COMMENT_ID) 可能是一个比较好的主键，因为这会把同一篇文章的评论存储在一起；
	- bigint 型的字段也是主键的可选项。
* 主键被隐式的连接到其他索引上：
	- KEY(A) 实际上是 KEY(A, ID);
	- Useful for sorting, Covering Index.

### 13. MySQL 怎么使用索引？
* 数据查询；
* 排序；
* Avoiding reading “data”；
* 特定优化。

### 14. 在数据查询中使用索引

1. 用了索引 LAST_NAME
	```MySQL
	SELECT * FROM EMPLOYEES WHERE LAST_NAME=“Smith”;
	```

2. 用了索引  (DEPT,LAST_NAME)
   ```MySQL
	SELECT * FROM EMPLOYEES WHERE
	LAST_NAME=“Smith” AND
	DEPT=“Accounting”
	```
	> 这里虽然索引字段顺序和查询的顺序颠倒，依然会走索引，不是因为最左匹配不走索引。
3. 多列索引会变的困难，对于索引 (A,B,C)：
	* 下面的条件会走索引:
		- A>5
		- A=5 AND B>6
		- A=5 AND B=6 AND C=7
		- A=5 AND B IN (2,3) AND C>5
	* 下面的条件不走索引，因为不符合最左匹配，缺少第一列
	   - B > 5
	   - B = 6 AND C = 7

	   > 在 MySQL5.7 中使用 explain 执行了一下，发现还是会走索引的，估计 MySQL 底层做了什么优化？

	* 下面条件会走部分索引
	 	- A>5 AND B=2
	 	- A=5 AND B>6 AND C=2

4. SQL 优化的第一原则：
