---
title: django mysql数据库读写分离
date: 2017-11-30 21:24:37
tags: django
---
- 前提

现在已经有了多个mysql数据库，并且**已经实现了主从同步**

- 问题

由于历史原因，项目在开发的时候并没考虑多数据库读写分离的情况，代码部分的查询更新都是使用了默认的default数据，代码大致如下

```python
    if not Model.object.filter(pk=xx).exists():
        Model.object.filter(xx=xx).update(yy=1)
```
此时如果要做读写分离，此时的exists等查询如果走了从库，由于并发or主从同步延迟，这个判断可能是错误的，导致了更新操作也是错误的

- 需要解决的问题

在不改原有代码的前提下，**更新操作前的查询必须使用主库**。
这里我们使用一个第三方库：[django-multidb-router](https://github.com/jbalogh/django-multidb-router)

- 使用方法

1.django mysql数据库中分别配置好主从数据库
```python
    DATABASES = {
        'default': {
            '...'
        },
        'slave': {
            '...'
        },
        'slave2': {
            '...
        }
    }
```

2.settings.py 的中插入该第三方库中间件, 置于第一个位置
```python
    MIDDLEWARE_CLASSES = (
        'multidb.middleware.PinningRouterMiddleware',
        '...'
    )
```

3.setting.py 新增从库list
```python
    SLAVE_DATABASES = ['salve', 'slave2', '...']
```

4.setting.py 新增数据库Router
```python
    DATABASE_ROUTERS = ('multidb.PinningMasterSlaveRouter',)
```

重点在第4步，该Router的源码如下
```python
class MasterSlaveRouter(object):
    """Router that sends all reads to a slave, all writes to default."""

    def db_for_read(self, model, **hints):
        """Send reads to slaves in round-robin."""
        return DEFAULT_DB_ALIAS if this_thread_is_pinned() else get_slave()

    def db_for_write(self, model, **hints):
        """Send all writes to the master."""
        return DEFAULT_DB_ALIAS

    def allow_relation(self, obj1, obj2, **hints):
        """Allow all relations, so FK validation stays quiet."""
        return True

    def allow_migrate(self, db, app_label, model_name=None, **hints):
        return db == DEFAULT_DB_ALIAS

    def allow_syncdb(self, db, model):
        """Only allow syncdb on the master."""
        return db == DEFAULT_DB_ALIAS
```
## 大致说明一下

1.数据库**写入**的时候，使用主库(默认default)
2.数据库**读取**的时候，判断该用户是否被标记过
    - 如果没有标记过，按顺序从SLAVE_DATABASES中取出一个从库读取数据
    - 如果被标记过，则直接从主库读取数据

## 重点如下

如果用户在指定时间内，有POST/PUT/DELET等请求，并且在这些请求中，通过python的[threading.local](https://docs.python.org/2/library/threading.html#threading.local)线程全局变量来标记该请求，
在发生数据库读取操作时，读取改变量的值，来判断是否读取主库

同时防止用户在调用POST/PUT/DELET等请求后，立刻查看结果，在这些请求结束后，上面配置的中间件会向浏览器写入一个15s(可配置)的cookie,
如果用户在请求中包含该cookie，则认为该请求被标记过(pinned)，读取主库确保用户可以看到正确的执行结果