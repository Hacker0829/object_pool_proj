## pip install object_pool

<pre style="color: darkgreen;font-size: medium">
python 通用对象池，socket连接池、mysql连接池归根结底都是对象池。
mysql连接池就是pymsql.Connection类型的对象池，一切皆对象。
只是那些很常用的功能的包都有相关的池库，池都是为他们特定的功能定制服务的，不够通用。

编码中很多创建代价大的对象（耗时耗cpu），但是他们的核心操作方法只能是被一个线程占用。

例如mysql，你用同一个conn在不同线程同时去高并发去执行插入修改删除操作就会报错，而且就算包不自带报错，
带事务的即使不报错在多线程也容易混乱，例如线程1要吧conn roallback，线程2要commit，conn中的事务到底听谁的。
解决类似这种抓狂的场景，如果不想再函数内部频繁创建和摧毁，那么就要使用池化思想。

</pre>

```

编码中有时候需要使用一种创建代价很大的对象，而且这个对象不能被多线程同时调用他的操作方法，

比如mysql连接池，socket连接池。
很多这样的例子例典型如mysql的插入，如果多线程高并发同时操作同一个全局connection去插入，很快就会报错了。
那么你可能会为了解决这个问题的方式有如下：

1.你可能这么想，操作mysql的那个函数里面每一次都临时创建mysql连接，函数的末尾关闭coonection，
  这样频繁创建和摧毁连接，无论是服务端还是客户端开销cpu和io高出很多。

2.或者不使用方案1，你是多线程的函数里面用一个全局connection，但是每一个操作mysql的地方都加一个线程锁，
  使得不可能线程1和线程2同时去操作这个connction执行插入，如果假设插入耗时1秒，那么100线程插入1000次要1000秒。

正确的做法是使用mysql连接池库。如果设置开启的连接池中的数量是大于100，100线程插入1000次只需要10秒，节省时间100倍。
mysql连接池已经有知名的连接池包了。如果没有大佬给我们开发mysql连接池库或者一个小众的需求还没有大神针对这个耗时对象开发连接池。
那么可以使用 ObjectPool 实现对象池，连接池就是对象池的一种子集，connection就是pymysql.Connection类型的对象，连接也是对象。
这是万能对象池，所以可以实现webdriver浏览器池。对象并不是需要严格实实在在的外部cocket或者浏览器什么的，也可以是python语言的一个普通对象。
只要这个对象创建代价大，并且它的核心方法是非线程安全的，就很适合使用对象池来使用它。




```

## 常问问题回答

### 1 对象池是线程安全的吗？

```
这个问题牛头不对马嘴 ，对象池就是为多线程或者并发而生的。
你想一下，如果你的操作只有一个主线程，那直接用一个对象一直用就是了，反正不会遇到多线程要使用同一个对象。

你花脑袋想想，如果你的代码是主线程单线程的，你有必要用dbutils来搞mysql连接池吗。
直接用pymysql的conn不是更简单更香吗。
web后端是uwsgi gunicorn来自动开多线程或者协程是自动并发的，虽然你没亲自开多线程，但也是多线程的，需要使用连接池。

任何叫池的东西都是为并发而生的，如果不能多线程安全，那存在的意义目的何在？

```

## 利用对象池来封装任意类型的池演示

contrib 文件夹自带演示了3个封装，包括http pymsql webdriver的池化。

以下是pymysql_pool的池化代码，使用has a模式封装的PyMysqlOperator对象，你也可以使用is a来继承方式来写，但要实现clean_up等方法。

```python
import copy

import pymysql
import typing
from universal_object_pool import ObjectPool, AbstractObject
from threadpool_executor_shrink_able import BoundedThreadPoolExecutor
import threading
import time
import decorator_libs

"""
这个是真正的用pymsql实现连接池的例子，完全没有依赖dbutils包实现的连接池。
比dbutils嗨好用，实际使用时候不需要操作cursor的建立和关闭。

dbutils官方用法是

pool= PooledDB()
db = pool.connection()
cur = db.cursor()
cur.execute(...)
res = cur.fetchone()
cur.close()  # or del cur
db.close()  # or del db

"""


class PyMysqlOperator(AbstractObject):
    error_type_list_set_not_available = []  # 有待考察，出了特定类型的错误，可以设置对象已近无效不可用了。

    # error_type_list_set_not_available = [pymysql.err.InterfaceError]

    def __init__(self, host='192.168.6.130', user='root', password='123456', cursorclass=pymysql.cursors.DictCursor, autocommit=False, **pymysql_connection_kwargs):
        in_params = copy.copy(locals())
        in_params.update(pymysql_connection_kwargs)
        in_params.pop('self')
        in_params.pop('pymysql_connection_kwargs')
        self.conn = pymysql.Connection(**in_params)

    """ 下面3个是重写的方法"""

    def clean_up(self):  # 如果一个对象最近30分钟内没被使用，那么对象池会自动将对象摧毁并从池中删除，会自动调用对象的clean_up方法。
        self.conn.close()

    def before_use(self):
        self.cursor = self.conn.cursor()
        self.core_obj = self.cursor  # 这个是为了operator对象自动拥有cursor对象的所有方法。

    def before_back_to_queue(self, exc_type, exc_val, exc_tb):
        if exc_type:
            self.conn.rollback()
        else:
            self.conn.commit()
        self.cursor.close()  # 也可以不要，因为每次的cusor都是不一样的。

    """以下可以自定义其他方法。
    因为设置了self.core_obj = self.cursor ，父类重写了__getattr__,所以此对象自动拥有cursor对象的所有方法,如果是同名同意义的方法不需要一个个重写。
    """

    def execute(self, query, args):
        """
        这个execute由于方法名和入参和逻辑与官方一模一样，可以不需要，因为设置了core_obj后，operator对象自动拥有cursor对象的所有方法，可以把这个方法注释了然后测试运行不受影响。
        :param query:
        :param args:
        :return:
        """
        return self.cursor.execute(query, args)


if __name__ == '__main__':
    mysql_pool = ObjectPool(object_type=PyMysqlOperator, object_pool_size=100, object_init_kwargs={'port': 3306})


    def test_update(i):
        sql = f'''
            INSERT INTO db1.table1(uname ,age)
        VALUES(
            %s ,
            %s)
        ON DUPLICATE KEY UPDATE
            uname = values(uname),
            age = if(values(age)>age,values(age),age);
        '''
        with mysql_pool.get(timeout=2) as operator:  # type: typing.Union[PyMysqlOperator,pymysql.cursors.DictCursor] #利于补全
            print(id(operator.cursor), id(operator.conn))
            operator.execute(sql, args=(f'name_{i}', i * 4))
            print(operator.lastrowid)  # opererator 自动拥有 operator.cursor 的所有方法和属性。 opererator.methodxxx 会自动调用 opererator.cursor.methodxxx


    operator_global = PyMysqlOperator()


    def test_update_multi_threads_use_one_conn(i):
        """
        这个是个错误的例子，多线程运行此函数会疯狂报错,单线程不报错
        这个如果运行在多线程同时操作同一个conn，就会疯狂报错。所以要么狠low的使用临时频繁在函数内部每次创建和摧毁mysql连接，要么使用连接池。
        :param i:
        :return:
        """
        sql = f'''
            INSERT INTO db1.table1(uname ,age)
        VALUES(
            %s ,
            %s)
        ON DUPLICATE KEY UPDATE
            uname = values(uname),
            age = if(values(age)>age,values(age),age);
        '''

        operator_global.before_use()
        print(id(operator_global.cursor), id(operator_global.conn))
        operator_global.execute(sql, args=(f'name_{i}', i * 3))
        operator_global.cursor.close()
        operator_global.conn.commit()


    thread_pool = BoundedThreadPoolExecutor(20)
    with decorator_libs.TimerContextManager():
        for x in range(200000, 300000):
            thread_pool.submit(test_update, x)
            # thread_pool.submit(test_update_multi_threads_use_one_conn, x)
        thread_pool.shutdown()
    time.sleep(10000)  # 这个可以测试验证，此对象池会自动摧毁连接如果闲置时间太长，


``` 

