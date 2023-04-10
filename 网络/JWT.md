# 认证方案

## Session 实现

登录认证后，通过服务端生成 Session，并将对应的 SessionId 作为 Cookie 发送给客户端。只要客户端每次请求时都携带 SessionId 的 Cookie，服务端如果通过该 SessionId 找到对应的 Session，意味着已经认证成功过。

![Session认证](https://raw.githubusercontent.com/Cavielee/notePics/main/Session认证.png)

缺点：

- 需要从内存或数据库里存取 session 数据
- 扩展性差，对于分布式架构，需要实现分布式 Session。



## JWT

![JWT认证](https://raw.githubusercontent.com/Cavielee/notePics/main/JWT认证.png)

认证通过后，服务端生成 JWT，JWT 包含用户数据等信息，并将 JWT 发送给客户端。客户端每次请求都会携带 JWT，服务端通过私钥进行验证，如果解析出来的公钥和自身公钥一致，则 JWT 有效，表明已经认证成功过。



# Jose

JOSE 全称 JSON Object Signing and Encryption，它定义了一系列的标准，**用来规范网络传输过程中使用 JSON 的方式**。

![JOSE](https://raw.githubusercontent.com/Cavielee/notePics/main/JOSE.png)

## JWA

JWA —— JSON Web Algorithms。 **JOSE 体系中涉及到的所有算法就是它来定义的**，比如通用算法有 Base64-URL 和 SHA，签名算法有 HMAC，RSA 和 Elliptic Curve（EC 椭圆曲线）。



## JWS

JWS —— JSON Web Signature。可以看出主要负责两个部分：

* **签名**：保证数据未被篡改。通过 JWA 提供的算法和密钥对数据进行加密生成。
* **验证**：检查签名是否被篡改。通过指定的密钥和 JWA 算法对签名进行解密，得出数据内容再和当前的数据内容进行比对，看是否被篡改。

缺点：如果服务端的密钥泄露的，那么任何客户端都能伪造 JWT。



## JWK

为了防止 JWS 中密钥容易泄露，导致 JWT 被伪造的问题，提出了 JWK。

JWK —— JSON Web Key ，它就是一个 JSON ，**JWK 就是用 JSON 来表示密钥**（JSON 字段因密钥类型而异）。

格式如下：

```json
{
    "e": "AQAB",
    "kty": "RSA",
    "n": "wVKQLBUqOBiay2dkn9TlbfuaF40_edIKUmdLq6OlvzEMrP4IDzdOk50TMO0nfjJ6v5830_5x0vRg5bzZQeKpHniR0sw7qyoSI6n2eSkSnFt7P-N8gv2KWnwzVs_h9FDdeLOeVOU8k_qzkph3_tmBV7ZZG-4_DEvgvat6ifEC-WzzYqofsIrTiTT7ZFxTqid1q6zrrsmyU2DQH3WdgFiOJVVlN2D0BuZu5X7pGZup_RcWzt_9T6tQsGeU1juSuuUk_9_FVDXNNCTObfKCTKXqjW95ZgAI_xVrMeQC5nXlMh6VEaXfO83oy1j36wUoVUrUnkANhp-dnjTdvJgwN82dGQ"
}
```

* kty 字段是必须的，代表密钥类型，支持 EC 椭圆曲线密钥，RSA 密钥和 oct 对称密钥。

上面例子为使用 RSA 非对称密钥：

* 服务端通过自己的私钥和 JWA 的算法对数据内容进行加密生成签名。
* 客户端对签名进行验证：
  * 通过服务端的公钥和 JWA 的算法对签名进行解密，将得出的数据内容和当前数据内容比较，判断数据内容是否有被篡改。

由于只有服务端才会持有私钥，而客户端只有公钥，因此可以确保只有服务端能生成 JWT，客户端只能使用公钥进行 JWT 的验证，无法生成 JWT。



## JWT

 JWT —— JSON Web Token。

> 可以通过网站 jwt.io 解析 JWT

![JWT案例](https://raw.githubusercontent.com/Cavielee/notePics/main/JWT案例.png)

JWT 分为三个部分：

* Header（头部）：JSON 对象，描述 JWT 的元数据。
  * alg 属性表示签名的算法（即具体 JWA 的算法），默认是 HMAC SHA256（简写为 HS256）；
  * typ 属性表示这个令牌（token）的类型（type），统一写为 JWT
* Payload（载荷）：JSON 对象，存放实际需要传递的数据，支持自定义字段
* Signature（签名）：生成签名去验证 JWT 是否未被篡改。



签名工作原理：

1. 将 `base64UrlEncode(Header)`和 `base64UrlEncode(Payload)` 用 `.` 连接成一个字符串。
2. 如果使用 JWS，则会使用密钥配合 Header 部分指定的 JWA 算法进行加密。
3. 如果使用 JWK，则会使用 JWK 配合 Header 部分指定的 JWA 算法进行加密。
   * 解密时，会根据 `Payload` 中的 `iss` ，然后从 `iss/.well-known/jwks.json` 获取 JWK，用 JWK 配合 Header 部分指定的 JWA 算法进行解密，判断 Header 和 Payload 内容是否被篡改，同时也用公钥判断 JWK 是否被篡改。
4. 最终使用base64UrlEncode对第二/第三步获取到的字符串进行加密生成签名。



## JWE

在 JWT 中，Header 部分和 Payload 部分内容是明文传输的，因此不适合传输敏感数据。

JWE —— JSON Web Encryption。 **JWE 的数据是经过加密的。它可以使 JWT 更加安全。**

JWE 提供了两种方案：共享密钥方案和公钥/私钥方案。

- Protected Header (受保护的头部) ：类似于 JWS 的 Header ，标识加密算法和类型。
- Encrypted Key (加密密钥) ：用于加密密文和其他加密数据的密钥。
- Initialization Vector (初始化向量) ：一些加密算法需要额外的（通常是随机的）数据。
- Encrypted Data (Ciphertext) (加密的数据) ：被加密的数据。
- Authentication Tag (认证标签) ：算法产生的附加数据，可用于验证密文内容不被篡改。



这五个部分的生成，也就是 JWE 的加密过程可以分为 7 个步骤：

1. 根据 Header alg 的声明，生成一定大小的随机数
2. 根据密钥管理方式确定 Content Encryption Key ( CEK )
3. 根据密钥管理方式确定 JWE Encrypted Key
4. 计算所选算法所需大小的 Initialization Vector (IV)。如果不需要，可以跳过
5. 如果 Header 声明了 zip ，则压缩明文
6. 使用 CEK、IV 和 Additional Authenticated Data ( AAD，额外认证数据 ) ，通过 Header enc 声明的算法来加密内容，结果为 Ciphertext 和 Authentication Tag
7. 最后按照以下算法构造出 Token：

```json
base64(header) + '.' +
base64(encryptedKey) + '.' + // Steps 2 and 3
base64(initializationVector) + '.' + // Step 4
base64(ciphertext) + '.' + // Step 6
base64(authenticationTag) // Step 6
```



## 总结

- JOSE：规范网络传输过程中使用 JSON 的一系列标准
- JWT：以 JSON 编码并由 JWS 或 JWE 安全传递的表示形式
- JWS：签名和验证 Token
- JWE：加密和解密 Token
- JWA：定义 JOSE 体系中涉及到的所有算法
- JWK：用 JSON 来表示密钥



# JWT 使用优化

由于 JWT 存在安全隐患，**下面是平时开发过程的一些建议：**

1. 始终执行算法验证

   签名算法的验证固定在后端，不以 JWT 里的算法为标准。假设每次验证 JWT ，验证算法都靠读取 Header 里面的 alg 属性来判断的话，攻击者只要签发一个 "alg: none" 的 JWT ，就可以绕过验证了。

2. 选择合适的算法

   具体场景选择合适的算法，例如分布式场景下，建议选择 RS256 。

3. HMAC 算法的密钥安全

   除了需要保证密钥不被泄露之外，密钥的强度也应该重视，防止遭到字典攻击。

4. 避免敏感信息保存在 JWT 中

   JWS 方式下的 JWT 的 Payload 信息是公开的，不能将敏感信息保存在这里，如有需要，请使用 JWE 。

5. JWT 的有效时间尽量足够短

   JWT 过期时间建议设置足够短，过期后重新使用 refresh_token 刷新获取新的 token 。