### Python 装饰器执行顺序

之前同事问到两个装饰器在代码中使用顺序不同会不会有什么问题，装饰器是对被装饰的函数做了一层包装，然后执行的时候执行了被包装后的函数，例如:

```python
def decorator_a(fun):
   def inner_a(*args, **kwargs):
       print 'inner_a'
       return fun(*args, **kwargs)
   return inner_a

@decorator_a
def test():
    print 'test_func'

test()
inner_a
test_func
```

在 Python 中函数也是一个对象，可以和其他对象一样当作一个参数传递，例如对于一个普通的函数，可以传递一些参数，然后在函数里对相应的参数进行操作，例如：

```python
def normal(param):
	print 'param:', param

normal('test')
param: test
type(normal)
function
```

而对于函数对象，可以如下进行传递：

```python
def get_func(f):
	print f.__name__
get_func(normal)
normal
```

所以上面使用的装饰器语法糖其实和以下语句功能一样，把 test 函数对象传给了装饰器函数，装饰器函数的返回值是 1 个函数对象，然后把函数对象赋给了和原先传进去的函数同名的变量，这样新的变量也就是一个函数对象了，对于函数对象，会和普通的函数一样。当然，使用 @ 会使代码看上去更加清晰：

```python
def test():
    print 'test_func'

test = decorator_a(test)
test()
```


上面是一个装饰器的情况，那么如果有多个装饰器，执行的顺序是怎么样的呢？下面是另一个装饰器：

```python
def decorator_b(fun):
     def inner_b(*args, **kwargs):
         print 'inner_b'
         return fun(*args, **kwargs)
     return inner_b
```
对于两个装饰器的执行顺序如下：

```python
# 1
@decorator_a
@decorator_b
def two():
	print 'two'

two()
inner_a
inner_b
two
# 2
@decorator_b
@decorator_a
def two():
	print 'two'

two()
inner_b
inner_a
two

# 3
two = decorator_b(decorator_a(two))
two()
inner_b
inner_a
two
```
可以看到，两个装饰器的执行顺序是从上往下执行的，对于多个装饰器，和单个装饰器类似，只是把上一个装饰器返回的函数对象又作为下一个装饰器的参数传递进去了而已。

到这里，也就大概明白了装饰器的执行顺序以及简单原理，回到开头，对于使用中的多个装饰器，还是需要关注一下使用的顺序的，例如一个验证是否登陆的装饰器，需要在验证用户身份的装饰器上面。大概如下：

```python
@login_in
@check_role
def hhh():
	pass
```
