Http消息(报文)结构如下所示：

```http
METHOD /url HTTP/1.1
Header1: Header1Value
Header2: Header2Value
Header3: Header3Value

Message Body
```

比如GET请求：

```http
GET /url HTTP/1.1
Host: localhost

```

无论有没有Message body，空格都是需要的。

如果有Message body，还需要指定Message的类型和长度，比如POST请求：

```http
POST /url HTTP/1.1
Host: localhost
Content-type: application/x-www-form-urlencoded
Content-length: 233

Your Message in length: 233
```

http的Method是请求类型，有8种请求类型：

- OPTIONS：返回服务器针对特定资源所支持的HTTP请求方法。也可以利用向Web服务器发送'*'的请求来测试服务器的功能性。
- HEAD：向服务器索要与GET请求相一致的响应，只不过响应体将不会被返回。这一方法可以在不必传输整个响应内容的情况下，就可以获取包含在响应消息头中的元信息。
- GET：向特定的资源发出请求。
- POST：向指定资源提交数据进行处理请求（例如提交表单或者上传文件）。数据被包含在请求体中。POST请求可能会导致新的资源的创建和/或已有资源的修改。
- PUT：向指定资源位置上传其最新内容。
- DELETE：请求服务器删除 Request-URI 所标识的资源。
- TRACE：回显服务器收到的请求，主要用于测试或诊断。
- CONNECT：HTTP/1.1 协议中预留给能够将连接改为管道方式的代理服务器。

比较常用的是GET / POST / HEAD, 其余不常用而且服务器不一定支持。

Http的响应消息结构如下所示：

```http
HTTP/1.1 StatusCode StatusDescription
Header1: Header1Value
Header2: Header2Value
Header3: Header3Value

Message Body
```

StatusCode的意义如下表：


分类 | 分类描述
---|---
1** | 信息响应，基本不用
2** | 成功，操作被成功接收并处理
3** | 重定向，需要进一步的操作以完成请求
4** | 客户端错误，请求包含语法错误或无法完成请求
5** | 服务器错误，服务器在处理请求的过程中发生了错误


常用状态码：

- 200: 请求成功 / 成功返回网页。一般用于GET与POST请求
- 301: 永久移动。请求的资源已被永久的移动到新URI，返回信息会包括新的URI，浏览器会自动定向到新URI。今后任何新的请求都应使用新的URI代替
- 302: 临时移动，客户端应继续使用原有URI
- 304: 未修改。所请求的资源未修改，服务器返回此状态码时，不会返回任何资源。客户端通常会缓存访问过的资源，通过提供一个头信息指出客户端希望只返回在指定日期之后修改的资源
- 307: 临时重定向。与302类似。把原本请求的数据POST到重新向
- 403: 服务器理解请求客户端的请求，但是拒绝执行此请求
- 404: 服务器无法根据客户端的请求找到资源（网页）。通过此代码，网站设计人员可设置"您所请求的资源无法找到"的个性页面
- 500: 服务器内部错误，无法完成请求
