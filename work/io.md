## 通过减少 IO 实现性能的优化

> 本文是工作中一点点简单的思考，不能保证是完全正确的，可能也仅仅是适用于部分场景。

场景 1：获取用户关注的好友列表中，每个好友的名字、头像等信息。

在很多公司中，不同的服务是由不同的人甚至是不同的部门维护的，这中间会通过一些定义好的接口进行交互（这里就用 RPC接口来说明了）。假设我们的服务是维护用户的关注关系，而用户的基本信息会维护在用户服务中。用户服务提供了如下两个接口：

```python
get_user_info(user_id):
	return {name='name', avatar='avatar'}

batch_get_user_info(user_ids):
	return map{user_id=user_info, user_id2=user_info}
```

假设我们提供给用户的关注好友列表，开始的时候是先获取用户的所有关注好友，然后依次去获取每个关注好友的信息，则提供给用户的接口 P95 可能会是 500ms，初始伪代码如下：

```python
def get_following_info():
	following_ids = [1, 2, 4, 5]
	user_infos = []
	for f_id in following_ids:
		user_infos.append(get_user_info(f_id))
	return user_infos
```
