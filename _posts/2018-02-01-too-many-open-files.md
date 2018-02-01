### 背景
我使用tornado做了一个实时日志查看的系统，远程日志查看的部分使用了redis的订阅和发布功能，
基本功能OK，不过，使用过程中出现了几次Too many open files...的错误，下面就是针对这一现象所做的总结。
### 初步解决办法
刚开始不太清楚具体的原因，以为是查看日志的人比较多，存在大量连接导致的。重启了实时日志服务后恢复正常了。而且由于系统默认的进程打开文件上限是1024，在大并发场景下显然是不够的，因此就改了
	这个上限值为10万，没有继续深究。
### 深入分析
后来，我做了个进程监控的工具，能在页面中展示服务器各个进程的详细信息，包括打开文件总数和具体的连接信息。我发现日志服务中存在大量的redis连接，而客户端的连接数并不多，这显然是不合理的。首先可以肯定的是我使用redis的方式有问题，于是我认真读了一下redis模块的代码，对发布订阅部分的实现限了解清楚。Python代码中的订阅方式是这样的：
```python
r = redis.StrictRedis(host=settings.REDIS_HOST, port=settings.REDIS_PORT,
                      password=settings.REDIS_PASSWD, db=5)
pubsub = r.pubsub(ignore_subscribe_messages=True)
# pubsub.subscribe(*args, **kwargs)
```
其中，一个pubsub对象会维护一个连接池，由于tornado是单线程的，因此，这个连接池只会有一个连接我在代码中是每一个请求就会实例化一次redis获取pubsub对象，因此，每一次请求都会新建立连接，
而之前的请求建立的连接在请求结束后就不再使用，由于代码中没有正确释放这些连接，导致连接数逐渐增多。事实上，因为tornado是单线程的，因此，整个服务器进程只需要维护一个redis连接就可以了，多余的连接都没有用处。所以，不需要在每次请求时去实例化redis，只需要在服务启动时实例化一个连接就够了，具体的做法就是上面的代码写到一个函数里，用一个全局变量保存调用结果就行了，这样，不管来多少请求，服务器端始终只有一个redis客户端(redis会在断开后自动重连，因此不用专门在每一次检测这个唯一的连接的有效性)。原来的方式是每一个请求对应一个redis客户端，所以，每一个客户端只需要订阅一个频道就行了，频道由服务器和日志路径唯一确定。现在的话，这个唯一的redis客户端就需要订阅多个频道了，之前的频道确定方法就不可行了。因为同一个日志会被多个用户查看，频道中的信息被某一个用户读取之后其他的用户就
读不到了。那么就可以再加上用户身份(remote_ip, remote_port)来确定唯一频道。不过问题又来了，原来多个用户同时读取一个日志文件时，只需要一个进程执行日志的实时读取和发送。现在的话，频道变了，即使是一个日志文件的查看也需要多个进程来执行了，一个频道对应一个进程。这样既浪费进程资源，也很浪费网络资源。是否有更好的解决方案呢？
有的，可以借鉴redis中订阅和发布的实现方式：
[订阅与发布](http://redisbook.readthedocs.io/en/latest/feature/pubsub.html)

redis-server中维护一个pubsub_channels 字典。key为频道，value为订阅这个频道的客户端列表。当有消息发送到频道上时，这条消息会被发送到频道对应的所有客户端。由于只有一个redis客户端，但是用户却有很多，因此，可以维护一个类似pubsub_channels的数据结构实现相同的功能。方法如下：还是根据服务器和日志确定唯一频道，如果有多个用户同时查看同一个日志，以channel为key，value为一个字典，保存所有的查看这个日志的用户。当收到频道中的消息时，根据这个channel找到所有的用户连接，发信息给这些用户，实现代码如下：
```python
from collections import defaultdict

from logs.utility import get_last_lines, get_pubsub


class SubWebSocket(tornado.websocket.WebSocketHandler):
	# 维护channel和用户请求的数据结构
	client_map = defaultdict(dict)

	def open(self, *args, **kwargs):
		print("opened")

	def assemble_cmd(self, log_path, cmd):
		kill_cmd = "kill `ps aux|grep logtail.py|grep %s|grep -v grep|awk '{print $2}'`" % (log_path,)
		return "{kill};{tail}".format(kill=kill_cmd, tail=cmd)

	@gen.coroutine
	def on_message(self, message):
		hostname, log_path, cmd = message.split("||")
		# 确保只有一个进程在实时读取及发布日志
		cmd = self.assemble_cmd(log_path, cmd)
		local = salt.client.LocalClient()
		channel = settings.LOG_KEY.format(
                server=hostname.strip(), log_path=log_path.strip())
        self.register(channel)
        if channel not in pubsub.channels:  # 避免重复订阅
	        pubsub.subscribe(**{channel: SubWebSocket.channel_callback})
	    local.cmd_async(hostname, "cmd.run", [cmd])
	    while True:
		    if not pubsub.connection.can_read(timeout=0):
			    yield gen.sleep(0.05)
			else:
				pubsub.get_message()

	@classmethod
	def channel_callback(cls, message):
		for client, callback in SubWebSocket.client_map[message["channel"]].iteritems():
			callback(message)

	def client_callback(self, message):
		line = format_line(message["data"])
		try:
			self.write_message(line)
		except tornado.websocket.WebSocketClosedError:
			self.unregister(message["channel"])

	def register(self, channel):
		client = self.request.connection.context.address
		SubWebSocket.client_map[channel][client] = self.client_callback

	def unregister(self, channel):
		client = self.request.connection.context.address
		SubWebSocket.client_map[channel].pop(client, None)
		if not SubWebSocket.client_map[channel]:
			SubWebSocket.client_map.pop(channel, None)
			pubsub.unsubscribe(channel)

	def on_close(self):
		print("closed")

```
目前来看，这种方式是最优的方式了，使用最少的连接，最少的频道，最少的进程以及最少的网络资源消耗。

完整的代码参见：
[github代码](https://github.com/tuxinhang1989/logs/blob/master/main.py)