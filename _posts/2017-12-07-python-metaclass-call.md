#### 元类是类的类，元类之于类就相当于类之于实例。
#### 元类的__new__方法会创建一个类并返回，就像类的__new__方法会创建一个实例并返回一样。
#### 元类中其他方法的定义类似于类中方法的定义，例如：
``` python
class Meta(type):
	def __new__(cls, name, bases, dct):  # cls为元类Meta
		return type.__new__(cls, name, bases, dct)

	def foo(cls, *args, **kwargs):  # cls为元类创建的类
		pass

	def __call__(cls, *args, **kwargs):  # cls为元类创建的类
		pass
```
***元类中有一个特殊的方法``__call__``，这个方法会阶段类的`__new__`和`__init__`***
#### `__call__` 应该返回实例，和类的`__new__`方法返回的一样。
### 下面看几个例子：
``` python
class Meta(type):
	def __new__(cls, name, bases, dct):
		print("calling Meta's __new__", cls)
		return type.__new__(cls, name, bases, dct)

	def __call__(cls, *args, **kwargs):
		print("calling Meta's __call__", cls)
		i = cls.__new__(cls)
		i.__init__(*args, **kwargs)
		return i

class A(object):
	__metaclass__ = Meta

	def __new__(cls, *args, **kwargs):
		print("calling A's __new__")
		return object.__new__(cls)

	def __init__(self, *args, **kwargs):
		print("calling A's __init__")

a = A()
print("a is", a)
```
#### 运行结果
``` python
calling Meta's __new__ <class '__main__.Meta'>
calling Meta's __call__ <class '__main__.A'>
calling A's __new__
calling A's __init__
a is <__main__.A object at 0x7faadaf02390>
```
#### 此时，还不太能看出来Meta.__call__拦截了`A.__new__`和`A.__init__`,只能看出，`__call__`会先于`__new__`和`__init__`调用。（其实可以看出，如果没有拦截发生，`calling A's __new__`和`calling A's __init__`会输出两次）

#### 为了更清楚的看出拦截行为，我们更改一下类的定义：
``` python
class Meta(type):
	def __new__(cls, name, bases, dct):
		print("calling Meta's __new__", cls)
		return type.__new__(cls, name, bases, dct)

	def __call__(cls, *args, **kwargs):
		print("calling Meta's __call__", cls)
		i = object.__new__(cls)
		i.__init__(*args, **kwargs)
		return i

class A(object):
	__metaclass__ = Meta

	def __new__(cls, *args, **kwargs):
		print("calling A's __new__")
		return object.__new__(cls)

	def __init__(self, *args, **kwargs):
		print("calling A's __init__")
```
#### 输出结果：
``` python
calling Meta's __new__ <class '__main__.Meta'>
calling Meta's __call__ <class '__main__.A'>
calling A's __init__
a is <__main__.A object at 0x7fd6b122c3d0>
```
#### 可以看出，`A.__new__`没有被调用，`__call__`返回的即为类的实例对象。
#### 如果去掉`__call__`中的`__init__`调用，上面输出结果中就不会出现`calling A's __init__`，由此可以确定，`__call__`拦截了`__new__`和`__init__`
## 注意：
**如果元类中定义了`__call__`，此方法必须返回一个对象，否则类的实例化就不会起作用。（实例化得到的结果为`__call__`的返回值）**

