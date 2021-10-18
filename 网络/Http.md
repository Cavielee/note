# 什么是 Http 协议

Http 协议是应用层协议。其传输层使用 TCP 协议，因此是一个可靠的协议。

HTTP（HyperText Transfer Protocol，超文本传输协议），实际是一种远程访问超文本资源的协议。



# URL

客户端需要访问服务端的资源，那么就需要一种东西来标识要访问的是服务端的什么资源？

这个时候就需要 URL（Uniform Resource Locator，统一资源定位符）。

在使用浏览器访问时，一般会输入如：

http://www.cavielee.com:80/java/index.html?name=cavie#search

上述地址实际上可以看成：

schema://host[:port]/path/.../?[url params]#[query string]

* scheme：指定应用层使用的协议，例如：HTTP、HTTPS、FTP
* host：HTTP 服务器的 IP 地址或者域名
* port：HTTP 服务器的默认端口是 80，这种情况下端口号可以省略。如果使用了其他端口则必须指明端口。
* path：访问资源的路径
* query string：查询字符串

上述这串地址就是标识了要访问的那个网络中的服务器的那个资源，即 URL。

> URI 用字符串标识某一互联网资源，而 URL 表示资源的地点（互联网上所处的位置）。可见 URL 是 URI 的子集。
>
> 实际上 URI 包含 URL 和 URN（Uniform Resource Name，统一资源名称），但一般 WEB 中使用的是 URL，因此一般将 URI 和 URL 近似相等。



# MIME TYPE

HTTP 是获取服务端超文本资源的协议，客户端需要对获取到的资源进行渲染，那么就必须知道资源的媒体类型（MIME Type）是什么。

常见的 MIME TYPE 如下：

* 文本文件：text /html、text/plain、text/css、application/xhtml+xml、application/xml
* 图片文件：image/jpeg、image/gif、image/png
* 视频文件：video/mpeg、video/quicktime

如何表示媒体类型？

HTTP 提供两种方式表示媒体类型：

* ACCEPT：表示客户端该请求可以接收的媒体类型。（服务端会生产客户端希望获得的媒体类型资源）
* Content-Type：表示服务端返回的资源具体的媒体类型。



# 请求和响应

HTTP 协议包含两个报文，请求报文 Request 和响应报文 Response。

客户端发起请求 Request，服务端收到后会对应的返回一个响应 Response。

## Request

![Request](C:\Users\63190\Desktop\pics\Request.png)

请求报文格式包含三个部分：请求行、请求头、请求体。

* 请求行：包括方法、URI、HTTP 协议版本

* 请求头：Host、Accept（可接收的媒体类型）、Accept-Language（可接收的语言类型）、Accept-Encoding（支持的压缩类型）、User-Agent（客户端内核）、Content-Length（请求体长度）
* 请求体：具体内容

> 请求头和请求体之间有一个空行分割。



## Response

![Response](C:\Users\63190\Desktop\pics\Response.png)

响应报文格式包含三个部分：状态行、响应头、响应体。

* 状态行：HTTP 协议版本、响应状态

* 响应头：Date（响应时间）、Server（服务端内核）、Last-Modified（资源最新修改时间）、ETag（和Last-Modified 配合使用来判断资源对象是否改变）、Accept-Ranges（范围请求的单位）、Content-Length（请求体长度）、Connection（连接类型）、Content-Type（响应资源类型）
* 响应体：返回资源

> 响应头和响应体之间有一个空行分割。



# 响应状态

响应报文中会有一个响应状态（状态码+描述），不同的状态码代表服务端不同的处理结果。

| 状态码 | 类别                             | 原因短语                   |
| ------ | -------------------------------- | -------------------------- |
| 1XX    | Informational（信息性状态码）    | 接收的请求正在处理         |
| 2XX    | Success（成功状态码）            | 请求正常处理完毕           |
| 3XX    | Redirection（重定向状态码）      | 需要进行附加操作以完成请求 |
| 4XX    | Client Error（客户端错误状态码） | 服务器无法处理请求         |
| 5XX    | Server Error（服务器错误状态码） | 服务器处理请求出错         |

常见状态码：

- **100 Continue** ：表明到目前为止都很正常，客户端可以继续发送请求或者忽略这个响应。
- **200 OK**：表示服务端正确处理了客户端发送过来的请求。
- **204 No Content** ：表示服务端正确处理请求，但没有报文实体要返回。
- **206 Partial Content** ：表示服务端正确处理了客户端的范围请求，并按照请求范围返回该指定范围内的实体内容。
- **301 Moved Permanently** ：永久性重定向，若之前的URI保存到了书签，则更新书签中的URI。
- **302 Found** ：临时重定向，该重定向不会变更书签中的内容。
- **303 See Other** ：临时重定向，与302功能相同，但是303状态吗明确表示客户端应当采用GET方法获取资源。
- **304 Not Modified** ：资源未变更，该状态码与重定向并没有什么关系，当返回该状态码时，告诉客户端请求的资源并没有更新，响应报文体中并不会返回所请求的内容。
- **307 Temporary Redirect** ：临时重定向，与 302 的含义类似，但是 307 要求浏览器不会把重定向请求的 POST 方法改成 GET 方法。
- **400 Bad Request** ：请求报文中存在语法错误。
- **401 Unauthorized** ：该状态码表示发送的请求需要进行HTTP认证（BASIC 认证、DIGEST 认证）。如果之前已进行过一次请求，则表示用户认证失败。
- **403 Forbidden** ：请求被拒绝，服务器端没有必要给出拒绝的详细理由。
- **404 Not Found**：找不到相应的资源，表示服务器找不到客户端请求的资源。
- **500 Internal Server Error** ：服务器内部错误，表示服务器在处理请求时出现了错误，发生了异常。
- **503 Service Unavilable** ：服务不可用，表示服务器处于停机状态，无法处理客户端发来的请求。

# 长连接和短连接

在 Http1.0 客户端和服务端之间的连接默认使用短连接，即每次请求前都需要通过 TCP 三次握手建立连接，并且在一次 HTTP 通信后会断开连接。

而实际使用中，客户端一般会进行多次请求访问，如果使用短连接的话，如果每进行一次 HTTP 通信就要断开一次 TCP 连接，连接建立和断开的开销会很大。

Http1.1 后默认使用长链接（客户端可以在请求中指定 Connection: Keep-Alive，表示建立长连接）。长连接表示客户端和服务端建立连接后，接下来的请求不需要重新建立连接，而是继续使用该连接，直到客户端请求或服务端响应设置 Connection: close 关闭客户端。



# 管道化连接

Http1.0 时客户端与服务端通信是传统阻塞式，即客户端要等待服务端对请求响应后，才能发送下一个请求。

Http1.1 后，支持管道化连接，使用管道可以并发的发送多个请求，不需要等待请求响应。



# 无状态

Http 是无状态协议，即协议本身并不会对请求和响应之间通信状态进行保存。

在一些请求中，需要用户进行登陆，由于 Http 无状态特性，因此用户登陆后，对于下一次继续访问这些请求时会重新要求用户进行登陆。这样重复登陆，会导致用户体验下降。因此需要一种机制，使得在使用 Http 协议时会保留状态。



## 客户端 Cookie

Http1.1 协议中引入了 cookie 技术，用来解决 http 协议无状态的问题。通过在请求和响应报文中写入 Cookie 信息来控制客户端的状态。

Cookie 会根据从服务器端发送的响应报文内的一个叫做 Set Cookie 的首部字段信息，通知客户端保存 Cookie。当下次客户端再往该服务器发送请求时，客户端会自动在请求报文中加入 Cookie 值后发送出去。

> Cookie 分为两类：
>
> - 会话期 Cookie：浏览器关闭之后它会被自动删除，也就是说它仅在会话期内有效。
> - 持久性 Cookie：指定一个特定的过期时间（Expires）或有效期（Max-Age）之后就成为了持久性的 Cookie。



**Set-Cookie 字段属性：**

| 属性         | 说明                                                         |
| ------------ | ------------------------------------------------------------ |
| NAME=VALUE   | 赋予 Cookie 的名称和其值（必需项）                           |
| expires=DATE | Cookie 的有效期（若不明确指定则默认为浏览器关闭前为止）      |
| path=PATH    | 将服务器上的文件目录作为 Cookie 的适用对象（若不指定则默认为文档所在的文件目录） |
| domain=域名  | 作为 Cookie 适用对象的域名（若不指定则默认为创建 Cookie 的服务器的域名） |
| Secure       | 仅在 HTTPs 安全通信时才会发送 Cookie                         |
| HttpOnly     | 加以限制，使 Cookie 不能被 JavaScript 脚本访问               |



### Cookie 禁用

为了防止服务端随便发送 Cookie 给客户端存储，客户端可以选择禁用 Cookie，那么此时如果要保留状态，如 SessionId，则客户端程序一般会进行 URL 重写，将 Cookie 数据附带在 URL 参数后，如 www.xxx.com/xxx?sid=xxx



## 服务端 Session

服务端提供 session 的机制来保存服务端的对象状态，服务器使用一种类似于散列表的结构来保存信息，通过唯一的 SessionId 来标识 Session。对于每一个连接服务端都会维护一个 Session。如果客户端请求没有携带 SessionId，服务端则会对应创建一个 Session，并将 SessionId 返回给客户端保存。如果客户端请求携带了 SessionId，则会根据 SessionId 查找对应的 Session，如果没找到则会创建新的 Session。

> 实际上 SessionId 会通过 Cookie 的形式返回给客户端，客户端之后请求都会带上 SessionId 的 Cookie。



# HTTP 方法

客户端请求报文中的请求行会有一个方法字段。

HTTP 提供以下方法

* GET：表示获取资源。服务器端接到GET请求后，就会明白客户端是要从服务器端获取相应的资源，然后就会根据请求报文中相应的参数，将需要的资源返回给客户端。使用GET方式的请求，传输的参数是拼接在URI上的。

* HEAD：表示获取响应头。服务端收到 HEAD 请求后，只会返回相应的响应头，不会返回响应体。

  和 GET 方法一样，但是不返回报文实体主体部分。主要用于确认 URL 的有效性以及资源更新的日期时间等。

* POST：客户端将数据放到请求体传输给服务端。

* PUT：上传文件，将文件内容放到请求体中，传输给服务器。因为HTTP/1.1的PUT方法自身不带验证机制，所以任何人都可以上传文件，存在安全性，所以上传文件时不推荐使用。

  由于自身不带验证机制，任何人都可以上传文件，因此存在安全性问题，一般不使用该方法。

* PATCH：对资源进行部分修改。

* DELETE：删除URI指定的资源。删除文件 DELETE用于删除URI指定的资源，与PUT一样，自身也是不带验证机制的。

* OPTIONS：查询支持的方法。会返回 Allow: GET, POST, HEAD, OPTIONS 这样的内容。

* CONNECT：要求在与代理服务器通信时建立隧道，实现用隧道协议进行TCP通信。主要使用SSL（Secure Sockets Layer, 安全套接层）和TLS（Transport Layer Security, 传输安全层）协议将通信内容进行加密后经网络隧道传输。

* TRACE：追踪路径。可追踪请求经过的代理路径，在发送请求时会为Max-Forwards头部字段填入数字，每经过一个代理中转Max-Forwards的值就会减一，直至Max-Forwards为零后，才会返回200。因为该方法易引起XST(Cross-Site Tracing，跨站追踪)攻击，所以不常用呢。



## RESTFul

RESTful 实际是一种规范，通过请求方法字段类型约束客户端的操作。

* GET：表示获取资源。

* POST：表示上传数据。

* PUT：表示内容的更新。

  由于自身不带验证机制，任何人都可以上传文件，因此存在安全性问题，一般不使用该方法。

* PATCH：对资源进行部分修改。

* DELETE：对应 API 的删除功能。



# Https

HTTP 是明文通信，因此请求和响应报文在通信过程中有以下安全性问题：

1. 报文被窃听，攻击者可以拦截报文进行窃听，然后再转发出去；
2. 攻击者可以作为中间人，伪装成客户端或者服务端；
3. 无法证明报文的完整性，报文有可能遭篡改。

为了解决 Http 通信的不安全性，提出了 Https 安全传输协议。

Https 是一种加密的超文本传输协议，与 HTTP 协议差异在于对数据传输的过程中，https 对数据做了完全加密。

Https 协议在 Tcp 协议层之上增加了一层 SSL（Secure Socket Laye）或者 TLS（Transport Layer Security）安全层传输协议组合使用，用于构造加密通道。

> TLS 实际上是 SSL 的升级版，实际上现在的 HTTPS 都是用的 TLS 协议，但是由于 SSL 出现的时间比较早，并且依旧被现在浏览器所支持，因此 SSL 依然是 HTTPS 的代名词 。



## 加密

SSL 是通过加密的形式构建通信通道。

加密方式有很多种：

### 对称密钥加密

对称密钥加密（Symmetric-Key Encryption），加密的加密和解密使用同一密钥。

优点：运算速度快；

缺点：密钥容易被获取。

[![img](https://github.com/crossoverJie/Interview-Notebook/raw/master/pics/7fffa4b8-b36d-471f-ad0c-a88ee763bb76.png)](https://github.com/crossoverJie/Interview-Notebook/blob/master/pics/7fffa4b8-b36d-471f-ad0c-a88ee763bb76.png)

#### 中间人攻击

服务端维护该密钥，客户端需要通过网络获取该密钥，然后通过该密钥进行加密请求报文，服务端根据该密钥进行解密。

但由于任何客户端都可以获取该密钥，因此攻击者可以获取该密钥，伪装成服务端或客户端进行通信，并可以窃取或篡改通信报文。即使每个客户端使用一个独立的密钥，也无法保证伪装服务端或客户端。

**解决方案：**

对于对称密钥的中间人攻击，可以采取人工传递密钥，并且每个客户端一个独立的密钥。如银行的U盾，客户端通过人工的方式获取服务端的密钥。



### 非对称密钥加密

公开密钥加密（Public-Key Encryption），也称为非对称密钥加密，使用一对密钥用于加密和解密，分别为公开密钥和私有密钥。接收端维护自己的私钥，并将公钥公开给其他发送端获取，发送端获取目标的公钥后进行加密发送，接收端使用自己的私钥对该密文进行解密。

- 优点：更为安全；
- 缺点：运算速度慢；

[![img](https://github.com/crossoverJie/Interview-Notebook/raw/master/pics/39ccb299-ee99-4dd1-b8b4-2f9ec9495cb4.png)](https://github.com/crossoverJie/Interview-Notebook/blob/master/pics/39ccb299-ee99-4dd1-b8b4-2f9ec9495cb4.png)

#### 中间人攻击

和对称加密一样，客户端和服务端无法识别中间伪装者，并不知道自己通信的是否是中间人。

下图为中间人攻击为将服务端的公钥篡改成自己的公钥发送给客户端。

![中间人攻击](C:\Users\63190\Desktop\pics\中间人攻击.jpg)



#### 第三方机构 CA

非对称密钥加密的中间人攻击实际上可以理解为客户端无法知道公钥是否为正真想要访问的服务端。

因此需要一个第三方权威机构存储服务端的公钥。

客户端会维护第三方机构的公钥（存储在浏览器中），客户端通过第三方机构获取想要通信的服务端公钥，第三方机构通过自己的私钥对该服务端公钥进行加密（数字证书）。

**证书伪装**

由于第三方机构的公钥是公开的，因此中间人可以通过该公钥将证书中的服务端公钥篡改成自己的公钥，导致客户端使用的是中间人的公钥。

**证书验证**

为了防止证书伪装问题，对数字证书中的服务端信息通过指定的加密算法（如 MD5）进行加密得到证书编号，该证书编号会存放在证书中（为了防止该证书编号被篡改，第三方机构会使用其私密对证书编号进行加密）。

客户端收到证书后，会根据证书中服务端信息的进行加密生成的证书编号，将该证书编号和证书中原由的证书编号（通过第三方机构公钥解密获得）进行对比，如果相同则表示证书没有被篡改。

![数字证书](C:\Users\63190\Desktop\pics\数字证书.jpg)

![数字证书验证](C:\Users\63190\Desktop\pics\数字证书验证.jpg)



> 实际上这个第三方机构就是数字证书的签发机构 CA



### HTTPs 加密方式

HTTPs 采用混合的加密机制，使用非对称密钥加密进行对称密钥传输，之后使用对称密钥加密进行通信。

客户端请求交互流程：

1. 客户端发起请求 Client Hello 包。
   1. 三次握手，建立 TCP 连接；
   2. 发送支持的安全协议版本 TLS/SSL；
   3. 客户端生成的随机数 client.random （用于生成后续通信的对称密钥）；
   4. 客户端支持的加密算法（用于后续的握手消息进行签名防止篡改）；
   5. sessionid ，用于保持同一个会话。

2. 服务端收到请求（Client Hello），然后响应 Server Hello 包。
   1. 确认加密通道协议版本（TLS/SSL）；
   2. 服务端生成的随机数 server.random（用于生成后续通信的对称密钥）；
   3. 确认使用的加密算法（用于后续的握手消息进行签名防止篡改）；
   4. 发送响应 Server Hello 包，包含服务器证书（ C A 机构颁发给服务端的证书）。

3. 客户端收到证书进行验证。
   1. 验证证书是否被篡改（验证证书编号）；
   2. 生成随机数 pre-master，并使用服务端的公钥进行加密（因此只有服务端才能解密获得）；
   3. 将 client.random、server.random、pre-master 进行计算获取对称密钥（用于后续通信使用）
   4. 使用约定好的 Hash 算法计算握手信息，并使用刚生成对称密钥加密。
   5. 将加密后的 pre-master 和加密握手信息发送给服务端。

4. 服务端接收随机数。
   1. 使用私钥解密获得 pre-master，并根据之前的 client.random 和 server.random 计算获得对称密钥，然后用该对称密钥解密获得通过客户端发送的 Hash 算法计算后的握手信息，并与自己用该 Hash 算法计算的握手信息值对比，验证对称密钥正确性。
   2. 根据握手信息生成数据并使用对称密钥发送给服务器。
5. 客户端接收消息
   1. 通过对称密钥解密获得握手信息数据，并验证。
   2. 至此正是建立通道，通过对称密钥进行通信。



总结：

通过非对称密钥通信建立通道，并交换随机数从而生成对称密钥（达到安全传输对称密钥），后面就是用对称密钥进行通信。



# GET 和 POST 的区别

## 参数

GET 和 POST 的请求都能使用额外的参数，但是 GET 的参数是以查询字符串出现在 URL 中，而 POST 的参数存储在内容实体中。

GET 的传参方式相比于 POST 安全性较差，因为 GET 传的参数在 URL 中是可见的，可能会泄露私密信息。并且 GET 只支持 ASCII 字符，如果参数为中文则可能会出现乱码，而 POST 支持标准字符集。

```
GET /test/demo_form.asp?name1=value1&name2=value2 HTTP/1.1
POST /test/demo_form.asp HTTP/1.1
Host: w3schools.com
name1=value1&name2=value2
```

## 安全

安全的 HTTP 方法不会改变服务器状态，也就是说它只是可读的。

GET 方法是安全的，而 POST 却不是，因为 POST 的目的是传送实体主体内容，这个内容可能是用户上传的表单数据，上传成功之后，服务器可能把这个数据存储到数据库中，因此状态也就发生了改变。

安全的方法除了 GET 之外还有：HEAD、OPTIONS。

不安全的方法除了 POST 之外还有 PUT、DELETE。

## 幂等性

幂等的 HTTP 方法，同样的请求被执行一次与连续执行多次的效果是一样的，服务器的状态也是一样的。换句话说就是，幂等方法不应该具有副作用（统计用途除外）。在正确实现的条件下，GET，HEAD，PUT 和 DELETE 等方法都是幂等的，而 POST 方法不是。所有的安全方法也都是幂等的。

GET /pageX HTTP/1.1 是幂等的。连续调用多次，客户端接收到的结果都是一样的：

```
GET /pageX HTTP/1.1
GET /pageX HTTP/1.1
GET /pageX HTTP/1.1
GET /pageX HTTP/1.1
```

POST /add_row HTTP/1.1 不是幂等的。如果调用多次，就会增加多行记录：

```
POST /add_row HTTP/1.1
POST /add_row HTTP/1.1   -> Adds a 2nd row
POST /add_row HTTP/1.1   -> Adds a 3rd row
```

DELETE /idX/delete HTTP/1.1 是幂等的，即便是不同请求之间接收到的状态码不一样：

```
DELETE /idX/delete HTTP/1.1   -> Returns 200 if idX exists
DELETE /idX/delete HTTP/1.1   -> Returns 404 as it just got deleted
DELETE /idX/delete HTTP/1.1   -> Returns 404
```

## 可缓存

如果要对响应进行缓存，需要满足以下条件：

1. 请求报文的 HTTP 方法本身是可缓存的，包括 GET 和 HEAD，但是 PUT 和 DELETE 不可缓存，POST 在多数情况下不可缓存的。
2. 响应报文的状态码是可缓存的，包括：200, 203, 204, 206, 300, 301, 404, 405, 410, 414, and 501。
3. 响应报文的 Cache-Control 首部字段没有指定不进行缓存。

## XMLHttpRequest

为了阐述 POST 和 GET 的另一个区别，需要先了解 XMLHttpRequest：

> XMLHttpRequest 是一个 API，它为客户端提供了在客户端和服务器之间传输数据的功能。它提供了一个通过 URL 来获取数据的简单方式，并且不会使整个页面刷新。这使得网页只更新一部分页面而不会打扰到用户。XMLHttpRequest 在 AJAX 中被大量使用。

在使用 XMLHttpRequest 的 POST 方法时，浏览器会先发送 Header 再发送 Data。但并不是所有浏览器会这么做，例如火狐就不会。



# 版本比较

## HTTP/1.0 与 HTTP/1.1 的区别

1. HTTP/1.1 默认是持久连接
2. HTTP/1.1 支持管线化处理
3. HTTP/1.1 支持虚拟主机
4. HTTP/1.1 新增状态码 100
5. HTTP/1.1 支持分块传输编码
6. HTTP/1.1 新增缓存处理指令 max-age



## HTTP/1.1 与 HTTP/2.0 的区别

### 多路复用

HTTP/2.0 使用多路复用技术，同一个 TCP 连接可以处理多个请求。

### 首部压缩

HTTP/1.1 的首部带有大量信息，而且每次都要重复发送。HTTP/2.0 要求通讯双方各自缓存一份首部字段表，从而避免了重复传输。

### 服务端推送

HTTP/2.0 在客户端请求一个资源时，会把相关的资源一起发送给客户端，客户端就不需要再次发起请求了。例如客户端请求 index.html 页面，服务端就把 index.js 一起发给客户端。

### 二进制格式

HTTP/1.1 的解析是基于文本的，而 HTTP/2.0 采用二进制格式。