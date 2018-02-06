### 通信模型：
salt 使用publish-subscribe模型进行通信，由salt-minion向salt-master建立连接。因此salt-minion不需要监听端口。
salt-master监听4505和4506端口。
* Publisher: (port 4505) 所有的salt-minions维持一个到Publisher的长连接。命令通过这些连接异步发送到minions，使得命令可以在所有minions上同时执行。
* Request Server: (port 4506) salt-minions在需要发送命令执行结果和请求文件时建立到Request Server的连接。

### Salt Minion认证：
* 当minion第一次启动时，它会在网络中搜索一个salt系统，找到之后，初始化连接并把自己的public key 发送给salt-master
* 在初始化连接之后，minion的public key就存储到salt-master了。master端必须接受这个key。在此之前，salt-minion不会执行任何命令。
* 在接受minion的public key之后。salt-master会返回自己的public key和用minion的public key加密的一个AES key(用于对传输数据进行加解密的AES key)

### 安全通信
之后的所有salt-master与salt-minions之间的通信都会用AES key进行加密。

### Rotating Security Keys
AES key 用于加密master->minion的jobs和minion->master的fileserver连接。每次重启salt-master或者删除salt-minion的key之后都会重新生成。

### 用户访问控制
在执行每个命令之前，salt会首先检查用户是否具有权限，如果没有权限，命令不会执行并报错。

### 远程执行
对于命令：salt '*' test.rand_sleep 120 执行流程如下：
* 通过Publisher port 将这个命令发送到所有连接着的minions
* 每个minion会检查自己是否符合target，从而决定是否执行此命令。
* 目标minion执行命令，然后将执行结果发送回Request Server

