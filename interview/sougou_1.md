> 简介：初级后端岗位，一面结束，失败。

先做了几个笔试题，面试开始会根据笔试题问一些内容。

#### 1. 一个代码题，要求输出的结果是什么，并解释原因</font>

```python
def f(x, l=[]):
  for i in xrange(x):
      l.append(i**2)
  print l

# 答案见注释
f(2)  # [0, 1]
f(3, [3, 2, 1])  # [3, 2, 1, 0, 1, 4]
f(3)  # [0, 1, 0, 1, 4]
```

简单解释：函数在初始化的时候，会把默认参数放在函数对象的 func_defaults 属性中，例如对于下面的 g 函数，可以看到 func_defaults 属性中会有默认参数的列表。对于 Python 函数中的默认参数，传递的是对象的引用，也就是多次调用的时候，默认参数如果不传，指向的是同一块内存，此时如果在函数内部更改了默认参数的值，会根据参数是否是可变对象有不同的反应。

对于不可变对象，会重新开辟一块内存，可以参考下面的输出可知（当然，对于一些小整数等，Python 是做了优化的，此时依然指向的是同一块内存，如果函数内部 b=2 的话，依然会指向同一块内存）；对于可变对象，则会在原来的内存上进行操作（当然，如果在函数内部进行某些操作，导致内存泄露又是另一个问题了），这会改变 func_defaults 中相应位置的参数的值。

在工作中，除非特殊情况，否则不要用可变对象作为默认参数，不然不知道啥时候会死的很惨的。

```python
In [54]: def g(a, b=2, c=[]):
    ...:     print 'idb:', id(b)
    ...:     print 'idc:', id(c)
    ...:     b='a'
    ...:     print 'idb2:', id(b)
    ...:     c.append(a)
    ...:     print 'idc2:', id(c)
    ...:

In [55]: g.func_defaults
Out[55]: (2, [])

In [56]: g(1)
idb: 140271775913920
idc: 4576730864
idb2: 4555404552
idc2: 4576730864

In [57]: id(g.func_defaults[0])
Out[57]: 140271775913920

In [58]: id(g.func_defaults[1])
Out[58]: 4576730864
```

#### 2. *args 和 **kwargs 是什么，为什么要使用它们？

简单解释：在定义函数或者类方法的时候，如果参数不确定，可以酌情使用他们。
前者可以认为是位置参数，会 pack 成 tuple, 只要传一些顺序的值就好了；
后者则是关键字参数，是一个 dict，需要传递 k = value 的形式。
对于 Python 中的参数、默认参数、*args 和 **kwargs 需要严格按照顺序使用。

（由于 Python 动态语言的灵活性，工作工程中，最好不要用这两种参数定义函数，
很多时候不知道传递的是个啥，需要传递什么，明明就定义了一个接口，还得去看内部的实现才知道怎么用，
当然开源的 Python 包中这种使用还是比较多的，例如著名的 [requests](https://github.com/requests/requests)，如果文档能写的这么完备，当我没说）。


#### 3. 说一下Python的内存管理，说一下垃圾回收机制？

*回顾总结*: 关于内存管理，个人觉得这篇比较好，[点击链接](http://www.cnblogs.com/vamei/p/3232088.html)，说的比较明白。
垃圾回收的三种机制是互相关联的，可以直接搜索找到合适自己的答案。

#### 4. 编程实现二分查找？

当时写的代码如下

```python
def b_search(data, k):
    if not data:
        return -1
    left, right = 0, len(data) - 1
    while left <= right:
        mid = (left + right) / 2
        if data[mid] == k:
            return mid
        elif data[mid] > k:
            right = mid - 1
        else:
            left = mid + 1
    return -1
```

追问，如果给定的数组中有多个重复值，如何进行查找？描述一下思路。当时理解成查找数组中这个值的始末索引了，回答的是使用二分法分别查找始末索引就可以了。面试官说直接使用二分法就可以。额？

#### 5. 经典问题，一次可以上1个台阶，也可以上2个...n个，问一共有多少种上法

```python
def fib(n):
    a, b = 1, 2
    for i in range(n):
        yield a
        a, b = b, 2 * b
```

#### 6. 当从b模块中导入a，运行b，会输出什么？a,b内容如下所示

```python
# a.py
print 'm'


def f():
    print 'fm'


class A(object):
    print 'A.m'

    def fa(self):
        print 'A.f.m'

# b.py

import a
```

运行b后，输入结果如下所示：
> m<br>
> A.m

解释：见 [这里](https://github.com/JushuangQiao/blog/blob/master/python/module_init.md)

#### 7.说一下对装饰器的理解，原理是什么？如何使用装饰器实现求函数的执行时间？

*回顾总结*：装饰器原理是使用了闭包实现的，是一种面向切面编程的模式（对设计模式理解不多，下一步需要抽时间学习设计模式了！）。对装饰器实现的执行时间，代码如下所示：（Python中有自带的timeit模块用来求执行时间）。

```python
import time
from functools import wraps


def timeit(func):
    @wraps(func)
    def wrapper():
        start = time.clock()
        func()
        end = time.clock()
        print 'used:', end - start

    return wrapper


@timeit
def f():
    time.sleep(3)

f()
```

#### 8.对闭包的理解？

*回顾总结*：闭包（Closure）是词法闭包（Lexical Closure）的简称，是引用了自由变量的函数。这个被引用的自由变量将和这个函数一同存在，即使已经离开了创造它的环境也不例外。

#### 9.对函数对象的理解？

*回顾总结*：函数可以赋值给变量、可以作为元素添加到集合对象中、可作为参数值传递给其它函数，还可以当做函数的返回值。对于函数对象的理解，可以看 [这里](https://foofish.net/function-is-first-class-object.html)

#### 10.如何理解Python中一切皆对象？

*回顾总结*：Python中所有的对象都有身份、类型和值。具体内容可以直接搜索，感觉这个 [博客](http://www.cnblogs.com/wangxin37/p/6598466.html) 写的也不错。当然要最好的理解还是看源码。

#### 11. 对协程的理解，协程和线程的区别？

*回顾总结*： 协程是使用生成器实现的，程序员自己来控制。协程适合 I/O 密集型操作，异步功能。

#### 12. 二叉树求深度？

递归思路

```python
def get_depth(tree):
    if not tree:
        return 0
    if not tree.left and not tree.right:
        return 1
    return 1 + max(get_depth(tree.left), get_depth(tree.right))
```

#### 13. MySQL 索引优化？

*回顾总结*：这个在很多面试中会问到，工作中也会用到，需要认真学好MySQL。关键点：最左匹配，联合索引，在哪种情况下建立索引，B 树原理，explain 查看语句等。网上搜一下挺多内容的，不过还是最好看一下《高性能MySQL》这本书。

#### 14. Redis 缓存，数据类型等？

*回顾总结*：字符串、列表、hash、set、有序集合等。
