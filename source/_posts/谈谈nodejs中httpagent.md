title: 谈谈nodejs中http.Agent
date: 2014-03-12 23:13:21
tags: 
- nodejs 
- http
---

http协议使用“你问-我答”的方式来传输数据，即客户端通过发送请求到服务端，服务端响应请求来完成交互。对每次交互都要经过”建立-传输-销毁“的过程，为了减少建立/销毁连接的开销，http提供了keep-alive模式，即“持久连接”对已经创建的连接可以重复使用，在浏览器中维护这些tcp连接是浏览器的行为我们不需要关心，但在服务端这些tcp连接需要我们自己去维护，还好node提供了http.Agent来帮我们维护.

node中通过http.Agent来管理这些可复用的连接，我们可以创建一个http.Agent实例
```
var agent = new http.Agent();
```

http.globalAgent是node默认创建的一个Agent实例，对http请求没有指定agent情况下，默认使用这个实例.

我们看看agent代理内部是如何管理tcp链接的。

```
Agent.prototype.addRequest = function(req, host, port, localAddress) {
  var name = host + ':' + port;
  if (localAddress) {
    name += ':' + localAddress;
  }
  if (!this.sockets[name]) {
    this.sockets[name] = [];
  }
  if (this.sockets[name].length < this.maxSockets) {
    // If we are under maxSockets create a new one.
    req.onSocket(this.createSocket(name, host, port, localAddress, req));
  } else {
    // We are over limit so we'll add it to the queue.
    if (!this.requests[name]) {
      this.requests[name] = [];
    }
    this.requests[name].push(req);
  }
};

```
agent通过将host、port和localAddress作为key用一个数组来存储可以复用的socket，
```
  self.on('free', function(socket, host, port, localAddress) {
    var name = host + ':' + port;
    if (localAddress) {
      name += ':' + localAddress;
    }

    if (!socket.destroyed &&
        self.requests[name] && self.requests[name].length) {
      self.requests[name].shift().onSocket(socket);
      if (self.requests[name].length === 0) {
        // don't leak
        delete self.requests[name];
      }
    } else {
      // If there are no pending requests just destroy the
      // socket and it will get removed from the pool. This
      // gets us out of timeout issues and allows us to
      // default to Connection:keep-alive.
      socket.destroy();
    }
  });

```
当socket完成一次请求后触发free事件，从request队列中进行下一次的请求，因此可以实现。


maxSockets：agent限制了对某个地址的最大请求数目，可以理解为浏览器限制对同一个域名端口的最大并发数.

为什么要进行限制呢？因为在linux平台上，操作系统为每个tcp连接都要创建一个socket句柄，而每个socket句柄同时也是个文件句柄，而操作系统对单一进程限制了最大可打开文件数量，通过ulimit -n命令可以查看当前操作系统限制创建文件数量，也可以用来设置最大可打开文件数。

```
#设置单进程最大可打开文件数为256
ulimit -n 256 
```

因此对一个线程可以创建的tcp连接有限，设置maxSockets做成防止对同一地址端口发出请求过大，我们可以通过修改maxSockets来设置最大连接数量.

```
http.globalAgent.maxSockets = 10; //设置最大同时请求数目为10个
```
http.Agent.defaultMaxSockets定义了默认的最大连接数为5。

默认情况下，http使用http.globalAgent来管理这些连接，我们可以通过传入agent参数来指定我们自己的agent。

```
var http = require('http');
var agent = new http.Agent();
var option = {
	hostname: 'www.qq.com',
	port: 80,
	method: 'get',
	path: '/',
	agent: agent
}
http.request(option, function(res) {
	// do something
});
```
也可以传agent为false来取消这个http请求的keep-alive的行为。
