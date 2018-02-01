### 大致思路：
	1.利用tornado提供的websocket功能与浏览器建立长连接，读取实时日志并输出到浏览器
	2.写一个实时读取日志的脚本，利用saltstack远程执行，并把实时日志发往redis中。
	3.tornado读取redis中的信息，发往浏览器。

#### 此过程用到了redis的发布和订阅功能。

### 先看一下tornado中是如何处理的：
```python
import os
import sys
import tornado.websocket
import tornado.web
import tornado.ioloop
import redis
import salt.client

from tornado import gen
from tornado.escape import to_unicode

from logs.utility import get_last_lines
from logs import settings


class SubWebSocket(tornado.websocket.WebSocketHandler):
	"""
	此handler处理远程日志查看
	"""
    def open(self, *args, **kwargs):
        print("opened")

    @gen.coroutine
    def on_message(self, message):
	    # 主机名，要查看的日志路径，运行脚本的命令这些信息从浏览器传过来
        hostname, log_path, cmd = message.split("||")
        local = salt.client.LocalClient()
        r = redis.StrictRedis(host=settings.REDIS_HOST, port=settings.REDIS_PORT,
                              password=settings.REDIS_PASSWD, db=5)
        # 订阅频道，服务器和日志路径确定一个频道
        key = settings.LOG_KEY.format(server=hostname.strip(), log_path=log_path.strip())
        channel = r.pubsub()
        channel.subscribe(key)
        # 异步方式执行命令，远程运行脚本
        local.cmd_async(hostname, "cmd.run", [cmd])
        try:
            while True:
                data = channel.get_message()
                if not data:
	                # 如果读取不到消息，间隔一定时间，避免无谓的CPU消耗
                    yield gen.sleep(0.05)
                    continue
                if data["type"] == "message":
                    line = format_line(data["data"])
                    self.write_message(line)
        except tornado.websocket.WebSocketClosedError:
            self.close()

    def on_close(self):
        global FLAG
        FLAG = False
        print("closed")


def format_line(line):
    line = to_unicode(line)
    if "INFO" in line:
        color = "#46A3FF"
    elif "WARN" in line:
        color = "#FFFF37"
    elif "ERROR" in line:
        color = "red"
    elif "CRITICAL" in line:
        color = "red"
    else:
        color = "#FFFFFF"

    return "<span style='color:{}'>{}</span>".format(color, line)


class EchoWebSocket(tornado.websocket.WebSocketHandler):
    def open(self):
        print("WebSocket opened")

    @gen.coroutine
    def on_message(self, message):
        log = message
        print "log file: ", log

        try:
            with open(log, 'r') as f:
                for line in get_last_lines(f):
                    line1 = format_line(line)
                    self.write_message(line1)
                while True:
                    line = f.readline()
                    if not line:
                        yield gen.sleep(0.05)
                        continue
                    self.write_message(format_line(line.strip()))
        except tornado.websocket.WebSocketClosedError as e:
            print e
            self.close()

    # def check_origin(self, origin):
    #     print origin, self.request.headers.get("Host")
    #     # super(EchoWebSocket, self).check_origin()
    #     return True

    def on_close(self):
        print("WebSocket closed")


class Application(tornado.web.Application):
    def __init__(self):
        handlers = [
            (r'/log/', MainHandler),  # 提供浏览页面，页面中的JS与服务器建立连接
            (r'/log/local', EchoWebSocket),  # 处理本地日志实时查看，比较简单
            (r'/log/remote', SubWebSocket),  # 处理远程日志实时查看，稍微复杂
        ]
        settings = {
            "debug": True,
            "template_path": os.path.join(os.path.dirname(__file__), "templates"),
            "static_path": os.path.join(os.path.dirname(__file__), "static"),
        }
        super(Application, self).__init__(handlers, **settings)


class MainHandler(tornado.web.RequestHandler):
    def get(self):
	    # 要查看的日志路径
        log = self.get_argument("log", None)
        # hostname实际上是saltstack中这台机器对应的minion id
        hostname = self.get_argument("hostname", None)
        # 本地日志还是远程日志
        type = self.get_argument("type", "local")
        # 运行读取实时日志的脚本，参数比较多，后面会有
        cmd = self.get_argument("cmd", "")
        context = {
            "log": log,
            "hostname": hostname,
            "type": type,
            "cmd": cmd,
        }
        self.render("index.html", **context)
```
### 配置文件中主要记录了redis服务器的地址等信息
```python
# encoding: utf-8

LOG_KEY = "logs:{server}:{log_path}"

LOG_NAME = "catalina.out"
TAIL_LINE_NUM = 20

REDIS_HOST = "127.0.0.1"
REDIS_PORT = "6379"
REDIS_PASSWD = None
REDIS_EXPIRE = 300

try:
    from local_settings import *
except ImportError:
    pass
```
### index.html的内容如下：
```html
<html>
<head>
<link href="{{ static_url('public/css/public.css') }}" rel="stylesheet" />
<link href="{{ static_url('kylin/css/style.css') }}" rel="stylesheet" />
</head>
<body style="background:#000000">
<div style="margin-left:10px;">
    <pre id="id-content">
    </pre>
    <div id="id-bottom"></div>
    <input type="hidden" id="id-log" value="{{ log }}" />
    <input type="hidden" id="id-type" value="{{ type }}" />
    <input type="hidden" id="id-hostname" value="{{ hostname }}" />
    <input type="hidden" id="id-cmd" value="{{ cmd }}" />
    <div class="btns btns_big">
        <button type="button" class="query_btn cancle" id="id-stop">Stop</button>
        <button type="button" class="query_btn commit" id="id-start">Start</button>
    </div>
</div>
<script type="text/javascript" src="{{ static_url('js/jquery-1.11.3.min.js') }}"></script>
<script type="text/javascript">
    var log_name = $("#id-log").val();
    var type = $("#id-type").val();
    var hostname = $("#id-hostname").val();
    var cmd = $("#id-cmd").val();
    // 初始化websocket对象
    var ws = new WebSocket("ws://{{ request.host }}/log/" + type);
    ws.onopen = function(){
        if (type === "local"){
            ws.send(log_name);
        } else {
	        // 建立连接后把相关信息发往服务器，对应上面的SubWebSocket
            ws.send(hostname + "||" + log_name + "||" + cmd);
        }
    };
    var get_message = function(evt){
        $("#id-content").append(evt.data + "\n");
        document.getElementById("id-bottom").scrollIntoView()
    };
    ws.onmessage = get_message;
    // 两个按钮控制日志的输出，如果看到需要的日志信息，可以暂停日志的输出，
    // 之后可以继续启动日志的输出
    $("#id-stop").click(function(){
        ws.onmessage = function(){};
    })
    $("#id-start").click(function(){
        ws.onmessage = get_message;
    })
</script>
</body>
</html>
```
### 这个tornado仅仅是提供了实时日志的服务，实际项目使用的是django，django中要做的其实很简单，提供log_name,hostname,type,cmd等四个参数。下面看一个实例：
```python
class LogView(KylinView):
	# 实时读取日志的脚本，事先使用saltstack批量传到各台服务器上
    client_path = "/tmp/logtail.py"

    def get(self, request):
        minion_id = request.GET.get("minion_id")
        context = {
            "minion_id": minion_id,
            "tail_log_url": settings.TAIL_LOG_URL,
        }
        return render(request, "cmdb/log_view.html", context)

    def post(self, request):
        minion_id = request.POST.get("minion_id")
        log_path = request.POST.get("log_path")
        if not log_path:
            return JsonResponse({"success": False, "message": "请填写日志路径"})
        try:
	        # 制定一开始读取的行数
            line_count = request.POST.get("line_count")
        except (TypeError, ValueError):
            return JsonResponse({"success": False, "message": "请输入正确的行数"})
        local = salt.client.LocalClient()
        # 确保saltstack能连通并且日志文件存在
        ret = local.cmd(minion_id, "file.file_exists", [log_path])
        if minion_id not in ret:
            return JsonResponse({"success": False, "message": "服务器无法连通"})
        if not ret[minion_id]:
            return JsonResponse({"success": False, "message": "日志文件不存在"})
        # 组成命令的各个参数，redis信息需要和tornado配置文件中的redis信息一致
        cmd = "{} {} {} {} {} {} {} {}".format(
            settings.PYTHON_BIN, self.client_path, minion_id, log_path, line_count, settings.REDIS_HOST,
            settings.REDIS_PORT, settings.REDIS_PASSWD)
        # settings.TAIL_LOG_URL是tornado中MainHandler对应的url,把其它几个
        # 参数组合成最终的URL，直接访问这个URL就可以在浏览器中实时读取日志了。
        url = "{}?type=remote&log={}&hostname={}&cmd={}".format(
            settings.TAIL_LOG_URL, log_path, minion_id, cmd)
        # 这一步的操作确保同一个日志文件只有一个脚本在读取，避免日志信息重复，这一步
        # 也很重要，必不可少
        local.cmd(minion_id, "cmd.run",
                  ["kill `ps aux|grep logtail.py|grep %s|grep -v grep|awk '{print $2}'`" % (log_path,)])
        return JsonResponse({"success": True, "url": url})
```

### 下面来看看logtail.py的实现：
```python
# encoding: utf-8
from __future__ import unicode_literals, division

import math
import time
import sys
import socket
import signal
import redis

FLAG = True


def get_last_lines(f, num=10):
	"""读取文件的最后几行
	"""
    size = 1000
    try:
        f.seek(-size, 2)
    except IOError:  # 文件内容不足size
        f.seek(0)
        return f.readlines()[-num:]

    data = f.read()
    lines = data.splitlines()
    n = len(lines)
    while n < num:
        size *= int(math.ceil(num / n))
        try:
            f.seek(-size, 2)
        except IOError:
            f.seek(0)
            return f.readlines()[-num:]
        data = f.read()
        lines = data.splitlines()
        n = len(lines)

    return lines[-num:]


def process_line(r, channel, line):
    r.publish(channel, line.strip())


def sig_handler(signum, frame):
    global FLAG
    FLAG = False


# 收到退出信号后，以比较优雅的方式终止脚本
signal.signal(signal.SIGTERM, sig_handler)
# 为了避免日志输出过多，浏览器承受不住，设置5分钟后脚本自动停止
signal.signal(signal.SIGALRM, sig_handler)
signal.alarm(300)


def get_hostname():
    return socket.gethostname()


def force_str(s):
    if isinstance(s, unicode):
        s = s.encode("utf-8")
    return s


def tail():
    password = sys.argv[6]
    if password == "None":
        password = None
    r = redis.StrictRedis(host=sys.argv[4], port=sys.argv[5], password=password, db=5)
    log_path = sys.argv[2]
    line_count = int(sys.argv[3])
    # 往redis频道发送实时日志
    channel = "logs:{hostname}:{log_path}".format(hostname=sys.argv[1], log_path=log_path)

    with open(log_path, 'r') as f:
        last_lines = get_last_lines(f, line_count)
        for line in last_lines:
            process_line(r, channel, force_str(line))
        try:
            while FLAG:  # 通过信号控制这个变量，实现优雅退出循环
                line = f.readline()
                if not line:
                    time.sleep(0.05)
                    continue
                process_line(r, channel, line)
        except KeyboardInterrupt:
            pass
    print("Exiting...")

if __name__ == "__main__":
    if len(sys.argv) < 6:
        print "Usage: %s minion_id log_path host port redis_pass"
        exit(1)

    tail()

```
到此为止，整个实时读取远程日志的流程就讲完了。
github: https://github.com/tuxinhang1989/logs