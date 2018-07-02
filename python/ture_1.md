开始的时候，需要用以下函数来做一个判断，根据返回的值来做一些后续判断处理：

```python
def is_success(param):
	if not param:
		return False
	return True

def process():
	ret = is_sucess('p')
	if ret:
		print 'success'
	else:
		print 'failed'

process()
```

  后来改需求了，要把失败时返回的结果更细化一些，以便调用的时候可以提示更多内容，而不是单纯的提示错误，于是把上面的函数做了如下改动

```python
def is_success(param):
	if param == 0:
		return 0
	elif param == 1:
		return 1
	return True

def process():
	ret = is_sucess('p')
	if ret == 0:
		print 'failed-0'
	elif ret == 1:
		print 'failed-1'
	else:
		print 'success'

process()
```

  结果调用的时候，问题来了，期待打印的‘success'没有出现，而是打印出了了‘failed-1'，也就是返回的 True 和 1 是相等的了，事实是这样吗？为什么呢？

  在 ipython 中执行了一下，发现还真是，如下

```python
In [1]: True == 1
Out[1]: True
```

  查看一下文档，发现 bool 是 int 的子类，并且仅有 True 和 False 两个实例，这也就解释了为什么 True == 1 为真了。

```python
class bool(int)
 |  bool(x) -> bool
 |
 |  Returns True when the argument x is true, False otherwise.
 |  The builtins True and False are the only two instances of the class bool.
 |  The class bool is a subclass of the class int, and cannot be subclassed.
 |
 |  Method resolution order:
 |      bool
 |      int
 |      object
 |
 |  Methods defined here:
 |
 |  __and__(...)
 |      x.__and__(y) <==> x&y
```

  既然 bool 是 int 的子类，True 和 Fasle 也就可以进行算术运算了，测试了一下，果然如此！

```python
In [3]: True + 2
Out[3]: 3

In [4]: True + True
Out[4]: 2

In [5]: False * 3
Out[5]: 0

In [6]: 4 / False
---------------------------------------------------------------------------
ZeroDivisionError                         Traceback (most recent call last)
<ipython-input-6-f1cbcd91af4b> in <module>()
----> 1 4 / False

ZeroDivisionError: integer division or modulo by zero
```

  其实 True 和 False 在 Python2 中是内建变量，既然是变量，也就可以和别的变量一样进行赋值了，其实这挺坑的，好在 Python3 中改成了关键字

```python
# python2
In [14]: import __builtin__

In [15]: dir(__builtin__)[41]
Out[15]: 'True'

In [16]: True=233

In [17]: True
Out[17]: 233

In [22]: import keyword

In [23]: keyword.iskeyword(True)
Out[23]: False

In [24]: keyword.iskeyword('True')
Out[24]: False
```

  在 Python3 中，True 被改成关键字了,这样就不能随便赋值导致一些潜在问题了

```python
>>> import keyword
>>> keyword.iskeyword('True')
True
>>> True=2
  File "<stdin>", line 1
SyntaxError: can't assign to keyword
```