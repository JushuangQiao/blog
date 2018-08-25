> 前提：由于业务需要，需要每一天扫描一个千万行的 MySQL 数据表，并用获取到的数据做一些事情，怎么能尽快的完成表的扫描呢？

-
一个思路：使用了批量 + 并发 + 异步的方法。
>
1. 首先获取到表中数据最大的 ID 和最小的 ID，并且设置一个阈值，例如 1000；
2. 建立一个异步批量分发函数，传入最大和最小的 ID；
3. 如果传入的最大和最小的 ID 的差值不大于阈值，则直接去调用异步任务的执行函数去完成 ID 段之间的扫描；
4. 如果大于阈值，则调用一个普通的分发函数，传入最大和最小的 ID，当两个 ID 之差小于阈值，继续异步调用批量分发函数；
5. 重复执行 3、4，直到分发完成。

思路解释：
>
1. 获取数据库的自增 ID，由于 ID 上会有聚簇索引，所以获取两个 ID 之间的数据会非常快；
2. 使用异步分发，可以保证某个数据段执行有问题的时候，不会影响到别的数据段的数据执行；
3. 使用并发，可以保证要执行的数据最快的执行完成，充分利用机器资源；
4. 由于按照 ID 段来异步扫描数据表，任何一段距离的失败都不会对其他 ID 造成影响。

代码示例:

```python

from celery import Celery
from sqlalchemy import and_, func, select
from sqlalchemy import create_engine
from sqlalchemy.ext.declarative import declarative_base

# mysql 相关设置
engine = create_engine('mysql://root:@localhost/wechat')
conn = engine.connect()

Base = declarative_base()
Base.metadata.reflect(engine)
tables = Base.metadata.tables

# celery 配置示例
app = Celery('tasks', backend='redis://localhost', broker='redis://localhost')

MAX_MYSQL_READ_RECORDS = 1000


class UserDAO(object):

    table = tables.user

    @classmethod
    def get_max_id(cls):
        sql = sql_select([func.max(cls.table.c.id)])
        return conn.execute(sql).scalar()

    @classmethod
    def get_by_ids(cls, start, end):
        sql = sql_select().where(
            and_(cls.table.c.id > start,
                 cls.table.c.id <= end
                 )
        )
        return conn.execute(sql).fetchall()


@app.task
def batch_spread(start, end):  # 入口函数
    distance = end - start
    if distance > MAX_MYSQL_READ_RECORDS:  # 数据量大，继续进行数据分发
        step = MAX_MYSQL_READ_RECORDS
    else:
        dispatch.delay(start, end)  # 数据够了，进行 ID 段之间的批量处理
        return

    spread_to_spread(start, distance, step)


def spread_to_spread(start, end, step):  # 数据分发函数
    while start + step < end:
        batch_spread.delay(start, start + step)
        start += step

    if end >= start:
        batch_spread.delay(start, end)


@app.task
def dispatch(start, end):  # 根据 ID 字段去表中取出一部分数据进行操作
    users = UserDAO.get_by_ids(start, end)
    for user in users:
        name = user.name
        send_message(name)


def send_message(name):  # 具体需要做的业务方法
    pass


if __name__ == '__main__:':
    batch_spread.delay((UserDAO.get_min_id(), UserDAO.get_max_id()))
```
