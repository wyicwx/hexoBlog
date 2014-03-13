title: nodejs中用zlib解压gzip
date: 2014-03-13 18:15:27
tags:
- nodejs
---

为了减少网络传输数据量，http传输过程中会采用通用的压缩算法来压缩数据，gzip属于最常用的压缩算法。

在node中http模块并没有帮我们进行解压，我们需要手动去判断gzip。

```
var http = require('http');
var options = {
	hostname: 'www.qq.com',
	port: 80,
	method: 'get',
	headers: {
		'Accept-Encoding': 'gzip'
	}
}

http.request(options, handler);

function handler(responder) {
	// do something
}
```

我们设置http头Accept-Encoding为gzip告诉服务器我们支持gzip压缩，服务器收到请求后，返回responder头中content-encoding带有gzip表明返回的数据为gzip压缩过的内容，node中可以通过responder.headers['content-encoding']来判断服务器返回内容是否gzip压缩过。

```
function handler(responder) {
	if(responder.headers['content-encoding'].indexOf('gzip') != -1) {
		// 解压gzip
	}
}

```

已经可以判断服务器返回的是gzip压缩过的内容，接下来我们需要去解压，幸好node提供了zlib模块。

我们用zlib模块来解压gzip

```
var zlib = require('zlib');
var fs = require('fs');
var gunzipStream = zlib.createGunzip();
var toWrite = fs.createWriteStream('qq.html');
```

zlib.createGunzip是一个transform流，通过pipe我们可以很方便解压gzip

```
function handler(responder) {
	if(responder.headers['content-encoding'].indexOf('gzip') != -1) {
		responder.pipe(gunzipStream).pipe(toWrite);
	}
}
```




