# IOLoop

### 事件
	* NONE = 0
	* READ = _EPOLLIN
	* WRITE = _EPOLLOUT
	* ERROR = _EPOLLERR | _EPOLLHUP

```
_instance_lock = threading.Lock()
_current = threading.local()

@staticmethod
def instance():
	if not hasattr(IOLoop, "_instance"):
		# 假如有多个线程同时执行到这一步，此时，都去争抢锁
		with IOLoop._instance_lock:
			# 假如第一个线程获取锁之后实例化了一次IOLoop,
			# 释放锁之后下一个线程获取锁之后，如果没有下面的判断，
			# 会再次实例化IOLoop，因此这里需要第二次判断
			if not hasattr(IOLoop, "_instance"):
				IOLoop._instance = IOLoop()
	return IOLoop._instance
```

```
@staticmethod
def initialized(self):
	assert not IOLoop.initialized()
	IOLoop._instance = self
	
def install(self):
	assert not IOLoop.initialized()
	IOLoop._instance = self
	# 在使用IOLoop的子类（比如EpollIOLoop)时需要调用install
	# 在子类中调用install，self即为子类实例
```

```
@staticmethod
def current(instance=True):
	""" 返回当前线程的IOLoop
	IOLoop._current == threading.local()
	"""
	current = getattr(IOLoop._current, "instance", None)
```