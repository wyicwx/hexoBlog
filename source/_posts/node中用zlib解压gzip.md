title: nodejs中用zlib解压gzip
date: 2014-03-13 18:15:27
tags:
- nodejs
---

为了减少网络传输数据量，http传输过程中会采用通用的压缩算法来压缩数据，gzip属于最常用的压缩算法。

使用node的http模块发送请求时并没有帮我们进行解压，因此我们需要手动去判断gzip。

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

若是要让服务端支持gzip压缩可以先判断请求头的accept-encoding是否含有gzip

```
var fs = require('fs');
var http = require('http');
var zlib = require('zlib');

http.createServer(function(req, res) {
	var file = fs.createReadStream('./qq.html');
	var acceptEncoding = req.headers['accept-encoding'];
	if(acceptEncoding && acceptEncoding.indexOf('gzip') != -1) {
		var gzipStream = zlib.createGzip();
		// 设置返回头content-encoding为gzip
		res.writeHead(200, {
			"content-encoding": "gzip"
		});
		file.pipe(gzipStream).pipe(res);
	} else {
		res.writeHead(200);
		// 不压缩
		file.pipe(res);
	}
}).listen(8080);

```

使用curl测试一下
```
# 不带有Accept-Encoding:gzip 返回正常文本
curl localhost:8080
# 带Accept-Encoding:gzip 返回压缩过的文件
curl -H "Accept-Encoding:gzip" localhost:8080
```

通过zlib可以实现客户端和服务端对gzip的压缩和解压

在express中提供了compress的中间件用来处理gzip