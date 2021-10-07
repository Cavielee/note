### 什么是CSRF

CSRF（Cross-site request forgery），中文名称：跨站请求伪造。



### CSRF 攻击流程

假设有两个网站：`A.com` （购物网站）和 `B.com` （恶意网站）

1. 用户登录了 `A.com` ，`A.com` 放回用户登录凭证（如Access-token）
2. 用户点击进入了 `B.com`
3. `B.com` 内嵌 js 访问了 `A.com`（用户并不知道）
4. 步骤③访问时会自动带上 Access-token，此时 `B.com` 就达到了仿冒用户发送请求访问 `A.com`



总结：恶意网站仿冒用户向以登录的网站发起访问请求，并获得了用户的隐私等。



### CSRF 攻击案例

（一）Get 请求

某银行转钱的接口，且该接口为 Get 请求。

```
http://www.a.com/transfer/id/1/money/1000
```

此时恶意网站 `B.com` 只需要隐藏一个图片即可访问该接口，并仿冒用户来访问该请求

```
<img scr="http://www.a.com/transfer/id/1/money/1000">
```



（二）Post 请求

对于第一种的改进，访问转钱接口约束为 Post 请求（表单提交）。

恶意网站只需隐藏一个表单，并通过 JS 进行表单提交：

```html
<html>
　　<head>
　　　　<script type="text/javascript">
　　　　　　function steal()
　　　　　　{
          　　　　 iframe = document.frames["steal"];
　　     　　      iframe.document.Submit("transfer");
　　　　　　}
　　　　</script>
　　</head>

　　<body onload="steal()">
　　　　<iframe name="steal" display="none">
　　　　　　<form method="POST" name="transfer"　action="http://www.myBank.com/Transfer.php">
　　　　　　　　<input type="hidden" name="toBankId" value="11">
　　　　　　　　<input type="hidden" name="money" value="1000">
　　　　　　</form>
　　　　</iframe>
　　</body>
</html>
```





### CSRF攻击的本质原因

首先允许跨域访问。CSRF攻击是源于Web的隐式身份验证机制！Web的身份验证机制虽然可以保证一个请求是来自于某个用户的浏览器，但却无法保证该请求是用户批准发送的。CSRF攻击的一般是由服务端解决。



### CSRF 攻击防御

1. 使用 https 传输，防止数据被截取下来。


2. 尽量使用POST，限制GET

一些数据更新的接口应该使用 Post 请求。



3. Cookie Hashing

通过设置一个 Cookie，并把该 Cookie 值加密后隐藏在表单中。访问接口时，对该值进行验证。

缺点：由于 Cookie 可能会被 XSS 攻击（即通过 JS 来窃取 Cookie）。

改进：Cookie 提供了 httpOnly 的设置，限定 Cookie 只能通过 http 访问时携带，禁止通过 JS 获取。



4. 加验证码

在提交表单中添加验证码，需要用户去手动获取验证码，并输入对应的验证码作为表单参数传到服务端，服务端对该验证码进行校验。

缺点：降低了用户体验，不能所有的接口都使用改方案，只有一些像金钱等重要接口需要。



5. Token

* 用户访问某个表单页面。

* 服务端生成一个Token，放在用户的Session中，或者浏览器的Cookie中。

* 在页面表单附带上 Token 参数。

* 用户提交请求后， 服务端验证表单中的 Token 是否与用户 Session（或 Cookies）中的 Token 一致，一致为合法请求，不是则非法请求。

> 若放在 Cookie 中，则要注意设置 Cookie 为 HttpOnly，防止攻击者通过 XSS 攻击获取到 Cookie。