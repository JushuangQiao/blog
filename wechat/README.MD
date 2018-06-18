# 2. 微信小程序相关的内容

## 2.1 实现小程序的登录
   小程序可以通过官方提供的登录能力来获取用户身份的标示，具体文档可以参考 [这里](https://developers.weixin.qq.com/miniprogram/dev/api/api-login.html)，通过流程时序可以看到，对于需要和前端配合的服务端开发，主要实现的就是通过小程序提供的 code 换取用户的 openid 和 session_key，并用换取到的 openid 和 secret_key 作为自定义的登录态。分析后得知，作为小程序后端的开发，主要实现以下几部分内容:

 	* 提供一个 HTTP 接口，供小程序方使用，传递 code；<br>
 	* 换取用户身份标识；<br>
 	* 维护一个自定义的登录态；
 	* 需要存储用户的 openid，以备后续使用。

1. 提供给小程序一个 HTTP 接口，接口使用 [Tornado](http://www.tornadoweb.org/en/stable/#) 框架

    > 为了简化思路，下面代码都没有做各种异常处理！

```python
class LoginHandler(RequestHandler):

    def post(self):
        req_data = json.loads(self.request.body)

        js_code = req_data.get('js_code')

        # 这里是换取用户的信息
        user_info = get_user_info(js_code=js_code)

        openid = user_info['openid']
        session_key = user_info['session_key']
        user_uuid = str(uuid.uuid4())  # 暴露给小程序端的用户标示

        # 用来维护用户的登录态
        User.save_user_session(
            user_uuid=user_uuid,
            openid=openid,
            session_key=session_key
        )
        # 微信小程序不能设置cookie，把用户信息存在了 headers 中
        self.set_header('Authorization', user_uuid)

        # 存储用户信息
        User.save_user_info(open_id=openid)

        self.set_status(204)
```


2. 换取用户身份标示，直接使用 [Requests](http://docs.python-requests.org/zh_CN/latest/) 请求微信的相关接口，获取数据

```python
def get_user_info(js_code):

    req_params = {
        "appid": 'app_id',  # 小程序的 ID
        "secret": 'secret',  # 小程序的 secret
        "js_code": js_code,
        "grant_type": 'authorization_code'
    }
    req_result = requests.get('https://api.weixin.qq.com/sns/jscode2session',
                              params=req_params, timeout=3, verify=False)
```

3. 维护一个自定义的登录态，使用了 [Redis](http://www.redis.cn)

```python
class User(object):

    REDIS_EXPIRES = 7 * 24 * 60 * 60
    table = tables['user']

    @classmethod
    def save_user_session(cls, user_uuid, openid, session_key):
        user_session_value = {
            'openid': openid,
            'session_key': session_key
        }
        user_session_key = 'US:' + user_uuid
        with user_redis.pipeline(transaction=False) as pipe:
            pipe.hmset(user_session_key, user_session_value)
            pipe.expire(user_session_key, cls.REDIS_EXPIRES)
            pipe.execute()
```


4. 存储用户信息，以备后用，这里使用了 [MySQL](https://www.w3cschool.cn/mysql/)，ORM 使用的是 [SQLAlchemy](https://www.sqlalchemy.org)

```python
class User(object):
    table = tables['user']

    @classmethod
    def save_user_info(cls, open_id):
        # 存储用户
        sql = cls.table.insert().values(open_id=open_id)
```

SQL 语句

```SQL
CREATE TABLE `user` (
  `id` int(20) unsigned NOT NULL AUTO_INCREMENT,
  `open_id` varchar(32) NOT NULL COMMENT '用户 open_id',
  `created_at` timestamp NULL DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
  `updated_at` timestamp NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT '更新时间',
  PRIMARY KEY (`id`),
  KEY `idx_oid` (`open_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```
全部代码可以点击 [这里](https://github.com/JushuangQiao/Python-Demos/tree/master/wechat) 获取！

## 2.2 发送模板消息
在小程序的开发过程中，会存在模板消息的发送，具体文档见 [这里](https://developers.weixin.qq.com/miniprogram/dev/api/notice.html#%E6%A8%A1%E7%89%88%E6%B6%88%E6%81%AF%E7%AE%A1%E7%90%86)，模板消息的发送是和语言无关的，这里将简要写一下怎么用 Python 给用户发送模板消息。
通过文档可以知道，发送的时候，需要使用小程序的 access_token 以及用户提交的 form_id，这里实现小程序的发送也就主要分为三部分:
     　1. 获取小程序的 access_token;
     　2. 获取用户提交的 form_id;
     　3. 给用户发送模板消息。

1. 获取小程序的 access_token，由于失效期为 2 小时，为了避免每次发送的时候都要去请求接口获取，这里可以使用一个定时任务，定时的时间只需要少于两个小时就可以，获取到 access_token 后，存储在 Redis 中，这样在小程序中包括发送模板消息在内，只需要直接读取 Redis 的值就可以了。示例代码如下：

```python
def get_access_token():
    payload = {
        'grant_type': 'client_credential',
        'appid': 'appid',
        'secret': 'secret'
    }

    req = requests.get('https://api.weixin.qq.com/cgi-bin/token', params=payload, timeout=3, verify=False)
    access_token = req.json().get('access_token')
    redis.set('ACCESS_TOKEN', access_token)
```

2. 获取用户提交的 form_id，这里只需要提供一个接口给小程序就可以了，代码示例如下：

```python
class FormHandler(RequestHandler):

    def post(self):
        req_data = self.request.body
        req_data = json.loads(req_data)
        form_id = req_data.get('form_id')
        template_push(form_id)  # 使用消息进行模板推送
```


3. 发送模板消息

```python
def template_push(form_id):
    data = {
        "touser": 'openid',
        "template_id": 'template_id',
        "page": 'pages/index/index',
        "form_id": form_id,
        "data": {
            'keyword1': {
                'value': 'value1'
            }
        },
        "emphasis_keyword": ''
    }
    access_token = redis.get('ACCESS_TOKEN')
    push_url = 'https://api.weixin.qq.com/cgi-bin/message/wxopen/template/send?access_token={}'.format(access_token)
    requests.post(push_url, json=data, timeout=3, verify=False)
```

至此，用户就会收到消息了，详细代码点击 [这里](https://github.com/JushuangQiao/Python-Demos/tree/master/wechat)
