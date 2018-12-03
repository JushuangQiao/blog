## 模块初始化的时候，发生了什么？

>假设有一个 hello.py 的模块，当我们从别的模块调用 hello.py 的时候，会发生什么呢？

方便起见，我们之间在 hello.py 的目录下使用 ipython 导入了。

hello.py 的代码如下，分别有模块变量，函数，类变量，类的静态方法、类方法和实例方法。

```python
# hello.py

print 'module value'

module_a = 233


def f():
    print 'func name:', f.__name__


class DoClass(object):
    print 'do class'
    c_value = 88

    @classmethod
    def cf(cls):
        print 'cf', cls.cf

    @staticmethod
    def sf():
        print 'sf', DoClass.sf.func_name

    def f(self):
        print self.f
```

在 hello.py 的目录下，开启 ipython，查看结果。

```python
In [1]: import sys

In [2]: sys.modules['hello']
---------------------------------------------------------------------------
KeyError                                  Traceback (most recent call last)
<ipython-input-2-ec73143594c2> in <module>()
----> 1 sys.modules['hello']

KeyError: 'hello'
```

在还没有导入 hello 模块的时候，可以看到，此时系统中是没有这个模块的，让我们进行一次导入。

```python
In [3]: import hello
module value
do class

In [4]: sys.modules['hello']
Out[4]: <module 'hello' from 'hello.pyc'>
```

导入后，可以看到模块中的模块变量和类变量都直接执行了，并且可以看到此时系统中是有了 hello 这个模块了。此时可以看一下 hello 模块的属性：

```python
In [5]: dir(hello)
Out[5]:
['DoClass',
 '__builtins__',
 '__doc__',
 '__file__',
 '__name__',
 '__package__',
 'f',
 'module_a']
```

可以看到，模块有模块变量、函数以及类三个属性。此时，在 ipython 中再次导入 hello，查看结果：

```python
In [6]: import hello

In [7]:
```
发现此时什么都没有输出，<font color="red">这是因为模块只有第一次被导入的时候才会被执行。</font>其实，可以把导入模块作为生成一个新的对象，模块变量和类变量都是在模块对象初始化的时候执行的，而函数和类的方法，则不会初始化的时候执行，只有在调用的时候才会执行：

```python
In [7]: hello.f()
func name: f

In [8]: hello.module_a
Out[8]: 233

In [9]: hello.DoClass
Out[9]: hello.DoClass

In [10]: hello.DoClass.c_value
Out[10]: 88
```

那么，如果在你模块中导入了其他的模块，或者导入了其他模块的方法，又是怎样的呢？

```python
# hello.py
# -*- coding: utf-8 -*-
import math
from math import sqrt
```
在模块中导入 math 以及 math 的 sqrt 方法，然后导入 hello:

```python
In [1]: import hello

In [2]: dir(hello)
Out[2]:
['__builtins__',
 '__doc__',
 '__file__',
 '__name__',
 '__package__',
 'math',
 'sqrt']

In [3]: hello.math
Out[3]: <module 'math' from '/usr/local/Cellar/python/2.7.14_1/Frameworks/Python.framework/Versions/2.7/lib/python2.7/lib-dynload/math.so'>

In [4]: hello.math.sqrt(100)
Out[4]: 10.0

In [5]: hello.sqrt(1000)
Out[5]: 31.622776601683793
```
可以看到，导入 hello 后，在 hello 中导入的 math 以及 sqrt 都成了 hello 的方法，这和我们直接在模块中定义的变量是一致的。

>
>总结：不管在模块中导入其他的模块或者直接定义变量，函数以及类，挡在别的模块中导入该模块的时候，这些内容都将成为该模块的属性。这其实和 Python 中一切皆对象是保持一致的。
