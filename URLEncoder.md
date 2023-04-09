# 原由

对于 GET 请求，服务器接受参数是通过 param 传输，即将参数拼接在 URL 后：

```
www.baidu.com/test?param1=value1&param2=value2
```

由于 url 规定参数中不允许出现空格、中文、特殊字符（如&@%?等）。

为了避免传输参数无法识别，因此在传输前应当将参数进行转义。

通过 `URLEncoder` 将参数进行转义，`URLDecoder` 将转义字符串进行解析。

```java
String s = "%aaa?bbb@ccc";
String encode = URLEncoder.encode(s, StandardCharsets.UTF_8);
System.out.println(encode);
System.out.println(URLDecoder.decode(encode, StandardCharsets.UTF_8));
```