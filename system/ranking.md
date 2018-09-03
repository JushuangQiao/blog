### 要求：实现一个用户排行榜，用户数量有很多，排行榜存储的是用户玩游戏的分数，对排行榜的读取压力比较大，如何实现？

### 思路分析：
>
1. 实现排行榜，可以考虑使用 Redis 的 zset 结构；
2. 用户数量很多的话，需要了解 zset 最多能存储多少元素；
3. zset 中的 value 使用用户的 user_id，score 就是用户游戏的得分；
4. 读取压力大，可以考虑读写分离，master 上写用户的分数，多个 slave 上读取数据，用来展示；
5. 如果存在多个 slave，怎么实现负载均衡，避免所有的流量都打到一个从库上呢？

### 实现

#### 1. zset 最多能存储多少元素？TODO 需要调研

#### 2. 如何使用 zset 更新用户数据，获取用户分数和排名，获取排行榜前 100 名；

```python
# 更新用户数据
def update_user_data(user_id, score):
	redis_key = 'RANKING'
    master_redis.zadd(redis_key, score, user_id)  # 对 master 进行写入操作

# 获取用户分数和排名
def get_user_ranking(user_id):
    redis_key = 'RANKING'
    with slave_redis.pipeline() as pipeline:
        pipeline.zscore(redis_key, member_id)
        pipeline.zrevrank(redis_key, member_id)
        results = pipeline.execute()
        score = results[0] or 0
        ranking = results[1]
    return {
        'score': int(score),
        'ranking': ranking+1 if ranking is not None else None
    }

# 获取某个区间的用户排行
def get_rankings(start=0, end=100):
	redis_key = 'RANKING'
	return [{'user_id': item[0], 'score': item[1]} for item in slave_redis.zrevrange(redis_key, start, end, withscores=True, score_cast_func=int)]
```

#### 3. 怎么实现负载均衡？

用户游戏的排行榜读多写少，可以对 master 进行写入操作，然后多个 slave 进行读取操作。

```python
import random
from redis import StrictRedis

master_redis = StrictRedis.from_url(conf['redis']['master'])
slave_redis = StrictRedis.from_url(random.choice(conf['slaves'))
```
