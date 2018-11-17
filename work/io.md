## 通过减少 IO 实现性能的优化

> 本文是工作中一点点简单的思考，不能保证是完全正确的，可能也仅仅是适用于部分场景。

### 场景 1：获取用户关注的好友列表中，每个好友的名字、头像等信息。

在很多公司中，不同的服务是由不同的人甚至是不同的部门维护的，这中间会通过一些定义好的接口进行交互（这里就用 RPC接口来说明了）。假设我们的服务是维护用户的关注关系，而用户的基本信息会维护在用户服务中。用户服务提供了如下两个接口：

```python
get_user_info(user_id):
	return {name='name', avatar='avatar'}

batch_get_user_info(user_ids):
	return map{user_id=user_info, user_id2=user_info}
```

假设我们提供给用户的关注好友列表，开始的时候是先获取用户的所有关注好友，然后依次去获取每个关注好友的信息，则提供给用户的接口 P95 可能会是 300ms，初始伪代码如下：

```python
def get_following_info():
	following_ids = [1, 2, 4, 5]
	user_infos = []
	for f_id in following_ids:
		user_infos.append(get_user_info(f_id))
	return user_infos
```

我们知道，和 IO 比起来，一般来说，CPU 计算要快得多，所以，针对以上代码，我们可以使用批量接口，直接获取到所有的用户信息。

```python
def get_following_info():
	following_ids = [1, 2, 4, 5]
	return batch_get_user_info(following_ids)
```

使用了以上代码之后，可能接口的性能会提高到 100 ms，此外，由于网络 IO 的减少，接口的错误也会少了很多。这就是一个优化。

当然，如果一个用户关注了很多的好友，可能没办法一次获取到所有的好友消息，这时候就需要分页了。

### 场景2: 批量查询数据库数据
一般情况下，应用服务器和数据库存储并不是在同一台服务器上，如果要进行数据库的查询操作，很多时候就需要通过网络来进行。

对于 MySQL 来说，数据是按照主键来进行物理存储的，如果表中没有主键，则会设置一个默认的主键。二级索引都会查询到主键之后，再次回表查询。也就是说，根据主键，可以认为更快的进行数据的查询。对于下面的表，如果我们想进行表的扫描，获取到 user_id，则可以首先获取到表的最大和最小主键 ID，然后批量获取两个 ID 之间的数据。

```SQL
CREATE TABLE `user_name` (
  `id` int(11) unsigned NOT NULL AUTO_INCREMENT COMMENT '自增id',
  `user_id` bigint(20) NOT NULL COMMENT ' 用户 id',
  `user_name` varchar(64) NOT NULL COMMENT '姓名',
  PRIMARY KEY (`id`),
  UNIQUE KEY `unq_uid` (`user_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

select max(id) from `user_name`;
select min(id) from `user_name`;
```
