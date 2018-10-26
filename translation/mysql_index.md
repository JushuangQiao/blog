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
	```
	SELECT * FROM EMPLOYEES WHERE LAST_NAME=“Smith”;
	```

2. 用了索引  (DEPT,LAST_NAME)
   ```
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

	MySQL 在多列索引中，一遇到 （<,>,between）就会停止使用 key，然而能继续使用 key 直到 in 范围的右边。

### 15. 在排序中使用索引
1.  排序
	```
	SELECT * FROM PLAYERS ORDER BY SCORE DESC LIMIT 10
	```
	- 该 SQL 会使用建立在 SCORE 列上的 索引；
	- 如果排序的时候没有使用索引，将会导致非常耗时的文件排序；
	- 在排序中经常会考虑组合索引， 例如下面的 SQL 可以考虑(COUNTRY,SCORE) 索引：
	```
	SELECT * FROM PLAYERS WHERE COUNTRY=“US” ORDER BY 	SCORE DESC LIMIT 10
	```

2. 使用多列索引进行高效的排序，在排序中使用索引有很多的限制，对于 KEY(A，B）:
	* 下面排序会使用索引：
		- ORDER BY A：主列索引；
		- A=5 ORDER BY B：通过第一列过滤数据，第二列进行排序；
		- ORDER BY A DESC, B DESC：用相同的排序进行排序；
		- A>5 ORDER BY A：主列上进行查询和排序
	* 下面的语句不会使用索引：
		- ORDER BY B ：非主列索引排序；
		- A>5 ORDER BY B：第一列上使用范围，第二列进行排序；
		- A IN(1,2) ORDER BY B：第一列上用 IN；
		- ORDER BY A ASC, B DESC：两列的排列顺序不同。

3. 使用索引进行排序的规则
	* 两列的排列顺序不能不一致；
	* 非排序的列中索引部分只能用 =，in 也不能用。

### 16. 避免读数据

1. [覆盖索引](https://yq.aliyun.com/articles/62419):对特定的查询使用索引，而不是对索引类型使用索引；
2. 仅仅读取索引，而不是读数据：索引比数据小；
3. <font color=#FF4500>SELECT STATUS FROM ORDERS WHERE CUSTOMER_ID=123</font> 使用索引 KEY(CUSTOMER_ID,STATUS)；
4. 通过索引读取数据是有顺序的，而通过数据指针读取数据经常是随机的。

### 17. 特定优化

1. Min/Max 优化
	* 索引对 Min/Max 聚集函数有帮助，当然也只对这俩有作用；
		- <font color=#BA55D3>SELECT MAX(ID) FROM TBL</font>;
		- <font color=#FF4500>SELECT MAX(SALARY) FROM EMPLOYEE GROUP BY DEPT_ID</font>：
			- Will benefit from (DEPT_ID,SALARY) index;
			- “Using index for group-by”

2. 索引和 JOIN
	* 在 MySQL 中使用 join 会导致嵌套循环，例如下面语句会遍历 POSTS 表找到 Peter 的所有文章，然后根据找到的文章从 COMMENTS 表中找到该文章的索引评论；

		>
		SELECT * FROM POSTS,COMMENTS WHERE
		AUTHOR=“Peter” AND COMMENTS.POST_ID=POSTS.ID
	* 在简单查询的时候才会使用索引，例如上面的语句不会使用 POST.ID 这个索引；
	* 重新设计 join 中没法进行索引的语句也是非常重要的。

### 18. 表中存在多个索引
1. MySQL 中可以存在多个索引：会有索引合并；
2. <font color=#FF4500>SELECT * FROM TBL WHERE A=5 AND B=6</font>：该语句能分别使用在 A 和 B 上的索引，但是在 （A,B) 上建立索引是更好的；
3. <font color=#FF4500>SELECT * FROM TBL WHERE A=5 OR B=6</font>：该语句使用两个独立的索引，但不会使用在（A,B) 上建立的索引

### 19. 前缀索引
可以在索引最左边一列上建立前缀索引：

1. ALTER TABLE TITLE ADD KEY(TITLE(20));
2. 需要在 BLOB/TEXT 上建立索引；
3. 能显著的提升效率；
4. 不能被用作覆盖索引；
5. 选择合适的前缀长度是一个问题。

### 20. MySQL 5.6 新特性
1. 很多优化器上的改进：
	- 这些改进会使你的查询更加自动；
	- join_buffer_size 有了新的意义，意味着该值大于 32 M，会有比较好的效果；

2. 本文专注于索引设计上的实践，其中最重要的一个是 ICP（索引条件下推）。

### 21. 理解 ICP
1. 下推一些过滤条件到引擎层：NAME LIKE “%ill%” 不能使用 range 过滤；
2. 更加灵活的覆盖索引：一些过滤在引擎层就完成了；
3. 5.5 之前：在范围内就会读取全部的数据。

### 22. ICP 举例
1. <font color=#FF4500>SELECT A … WHERE B=2 AND C LIKE “%ill%’</font>
	- MySQL 5.5 and below：
		* KEY (B) – Traditional. Using index for range only
		* KEY (B,C,A) - 覆盖索引，用到的列都在索引中；
	- MySQL 5.6
		* KEY(B,C)：通过 B 找到数据范围; 会根据 C 读取所有的列查找满足条件的数据；
2. <font color=#FF4500>SELECT * …WHERE A=5 and C=6 ;</font>
	- KEY (A,B,C)：Will scan all index entries with A=5 not all rows。

### 23. 怎么选择使用哪一个索引？
1. 对于每一个查询动态的选择索引：查询文本中的常量非常重要；
2. 通过下潜到表中，估算每个索引需要问访问的列数；
3. 使用基数统计，这个会分析表的更新。

### 24. 用 EXPLAIN
EXPLAIN 是一个非常好的查看数据库怎么执行查询的工具：

- https://dev.mysql.com/doc/refman/5.5/en/using-explain.html；
- 需要记住真是的执行可能和 EXPLAIN 不同
- 5.6 中可以使用 json 格式化 EXPLAIN 的结果：
	*  <font color=#FF4500>EXPLAIN FORMAT=JSON SELECT * FROM t1 JOIN t2 ON t1.i = t2.i WHERE t1.j > 1 AND t2.j < 3;</font>

### 25. 索引策略
1. 对于关键的查询建立索引，需要整体来看索引，不要单独的看；
2. 最好的对于 WHERE 和 JOIN 的所有查询条件使用索引，至少也是大部分条件；
3. 多数情况下拓展现有索引，不要新建索引；
4. 作出改变的时候，验证一下性能的影响，把所有的查询都执行一遍。

### 26. 索引示例
1. 索引要有利于多个查询：
	- <font color=#48D1CC>SELECT * FROM TBL WHERE A=5 AND B=6</font>
	- <font color=#FF4500>SELECT * FROM TBL WHERE A>5 AND B=6</font>
	- KEY (B,A) Is better for such query mix
2. 把最有选择性的条件放在索引第一位；
3. 对于非关键查询，可以不增加索引，索引太多会影响性能。

### 27. 技巧
1. 假设索引的顺序是 KEY(A,B)
	- <font color=#FF0000>SELECT * FROM TBL WHERE A BETWEEN 2 AND 4 AND B=5</font>：只会用到索引的第一部分；
	- <font color=#48D1CC>SELECT * FROM TBL WHERE A IN (2,3,4) AND B=5</font>：会用到整个索引。

2. 增加 FAKE 过滤，例如 KEY(GENDER,CITY)，想要只使用一个索引：
	- <font color=#8470FF>SELECT * FROM PEOPLE WHERE CITY=“NEW YORK”</font>：不会用到索引；
	- <font color=#FF0000>SELECT * FROM PEOPLE WHERE GENDER IN (“M”,”F”) AND CITY=“NEW YORK”</font>：会使用索引；
	- 这个技巧对于低选择的列非常有用，例如性别、状态、布尔类型等。

	> 个人理解：对于 key，如果想使用后面的列，要想走索引的话，可以把前面的列全部列出来，当然，前面的列的可能数最好比较小。

3. 文件排序，例如只有 KEY(A,B):
	- <font color=#4169E1>SELECT * FROM TBL WHERE A IN (1,2) ORDER BY B LIMIT 5 </font>: 不会使用索引；
	- <font color=#FF0000>(SELECT * FROM TBL WHERE A=1 ORDER BY B LIMIT 5) UNION ALL (SELECT * FROM TBL WHERE A=2 ORDER BY B LIMIT 5) ORDER BY B LIMIT 5</font>：会用到索引，排序只会排列 10 行。

## 联系人
Email: pz@percona.com
