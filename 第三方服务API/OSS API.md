# 临时访问凭证

## 来源

对于 OSS 操作需要提供 OSS 密钥（AccessKey ID 和 AccessKey Secret）。

而实际开发中 OSS 密钥一般有服务端持有，而 APP 应用中有时候需要上传资源到 OSS，此时解决方案如下：

1. 将资源传输到服务端，由服务端同一上传。
2. APP 持有 OSS 密钥。

分析：

第一种方案中，服务端实际上并不关注资源上传的过程，而在于资源上传到 OSS 后的访问 URL，因此资源没必要经过一层服务端的传输。

第二种方案中，将 OSS 密钥暴露在 APP 应用中是不安全的。

因此 OSS 提供一种更安全更有效的方法：**临时访问凭证。**



## 工作流程

1. APP 客户端请求服务端获取临时访问凭证。
2. 服务端通过请求 STS 服务器去获取临时访问凭证（安全令牌（SecurityToken）+ 临时访问密钥（AccessKey Id 和 AccessKey Secret）+ 过期时间）并返回。

3. APP 客户端缓存该临时访问凭证，并在有效期内通过凭证访问 OSS API，OSS 会根据凭证信息到 STS 进行校验，检验通过则正常使用。



## 代码

定义授权策略：

```java
// 赋予角色向目标存储空间examplebucket下的目录exampledir上传文件的权限
String writePolicy = "{\n" +
    "    \"Version\": \"1\",\n" +
    "    \"Statement\": [\n" +
    "     {\n" +
    "           \"Effect\": \"Allow\",\n" +
    "           \"Action\": [\n" +
    "             \"oss:PutObject\"\n" +
    "           ],\n" +
    "           \"Resource\": [\n" +
    "             \"acs:oss:*:*:examplebucket/exampledir\",\n" +
    "             \"acs:oss:*:*:examplebucket/exampledir/*\"\n" +
    "           ]\n" +
    "     }\n" +
    "    ]\n" +
    "}";
```

请求临时访问凭证：

```java
public static void main(String[] args) { 
    // 角色ARN
    String roleArn = "myRoleArn";
    // 角色会话名称
    String roleSessionName = "app-oss-001";
    try {
        // 指定RAM地域ID、OSS密钥
        IClientProfile profile = DefaultProfile.getProfile("cn-shenzhen", oSSConfig.getAccessId(), oSSConfig.getAccessSecret());
        // 构造client
        DefaultAcsClient client = new DefaultAcsClient(profile);

        // 发起临时访问凭证请求
        final AssumeRoleRequest request = new AssumeRoleRequest();
        // 适用于Java SDK 3.12.0及以上版本。
        request.setSysMethod(MethodType.POST);
        // 适用于Java SDK 3.12.0以下版本。
        //request.setMethod(MethodType.POST);
        // 设置角色ARN
        request.setRoleArn(roleArn); 
        // 自定义角色会话名称，用来区分不同的令牌
        request.setRoleSessionName(roleSessionName);
        // 设置授权策略，如果policy为空，则用户将获得该角色下所有权限。
        request.setPolicy(policy);
        // 设置临时访问凭证的有效时间为3600秒。
        request.setDurationSeconds(3600L); 
        // 响应
        final AssumeRoleResponse response = client.getAcsResponse(request);
        System.out.println("Expiration: " + response.getCredentials().getExpiration());
        System.out.println("Access Key Id: " + response.getCredentials().getAccessKeyId());
        System.out.println("Access Key Secret: " + response.getCredentials().getAccessKeySecret());
        System.out.println("Security Token: " + response.getCredentials().getSecurityToken());
        System.out.println("RequestId: " + response.getRequestId());
    } catch (ClientException e) {
        System.out.println("Failed：");
        System.out.println("Error code: " + e.getErrCode());
        System.out.println("Error message: " + e.getErrMsg());
        System.out.println("RequestId: " + e.getRequestId());
    }
}
```

详情可以看：https://help.aliyun.com/document_detail/100624.html



# 创建 Bucket

创建 Bucket 可以理解为创建文件夹，文件夹用于存储资源。

```java
public void createBucket(String bucketName) {
    // 创建OSSClient实例。
    OSS ossClient = new OSSClientBuilder().build(oSSConfig.getEndpoint(), oSSConfig.getAccessId(), oSSConfig.getAccessSecret());

    try {
        // 创建存储空间。
        ossClient.createBucket(bucketName);

    } catch (OSSException oe) {
        System.out.println("Caught an OSSException, which means your request made it to OSS, "
                           + "but was rejected with an error response for some reason.");
        System.out.println("Error Message:" + oe.getErrorMessage());
        System.out.println("Error Code:" + oe.getErrorCode());
        System.out.println("Request ID:" + oe.getRequestId());
        System.out.println("Host ID:" + oe.getHostId());
    } finally {
        if (ossClient != null) {
            ossClient.shutdown();
        }
    }
}
```



# 上传文件

将指定的资源上传到 bucket。

资源名字中的 `/` 会自动创建对应文件夹

```java
public String upload(OssUploadDTO ossUploadDTO) {
    String uploadUrl = null;
    // 文件目录 年/月/日
    LocalDateTime now = LocalDateTime.now();
    String directory = now.getYear()+"/"+now.getMonthValue()+"/"+now.getDayOfMonth();
    // 获得资源输入流
    InputStream inputStream = new ByteArrayInputStream(ossUploadDTO.getUpload());
    // 创建OSSClient实例。
    OSS ossClient = new OSSClientBuilder().build(oSSConfig.getEndpoint(), oSSConfig.getAccessId(), oSSConfig.getAccessSecret());
    try {
        // meta设置请求头
        ObjectMetadata meta = new ObjectMetadata();
        meta.setContentType("image/jpg");
        // 设置路径+文件名
        // 如Account/2022/12/31/1.jpg，对应会在bucket生成Account目录-》2022目录...
        String objectName = ossUploadDTO.getContentType()+"/"+directory+"/"+snowflake.nextId()+"."+ossUploadDTO.getFileType();
        // 上传oss
        ossClient.putObject(oSSConfig.getBucket(), objectName, inputStream, meta);
        // 返回上传之后地址，拼接地址
        uploadUrl = oSSConfig.getMappingDomain()+"/"+objectName;

        //            //设置文件的访问权限为私有
        //            OSSSignResp ossSignResp = new OSSSignResp();
        //            ossSignResp.setBucket(oSSConfig.getBucket());
        //            ossSignResp.setObjectName(objectName);
        //            setObjectPrivate(ossSignResp);
    } catch (Exception e) {
        log.error("upload oss error", e);
    } finally {
        if(null!=ossClient){
            ossClient.shutdown();
        }
        try {
            if(null!=inputStream){
                inputStream.close();
            }
        } catch (IOException e) {
            log.error("upload oss error", e);
        }
    }
    return uploadUrl;

}
```

# 设置文件读写权限

默认上传的文件读写权限是根据 Bucket 定义的文件读写权限。文件读写权限如下：

- **继承Bucket**：以Bucket读写权限为准。
- **私有**（推荐）：只有文件Owner拥有该文件的读写权限，其他用户没有权限操作该文件。
- **公共读**：文件Owner拥有该文件的读写权限，其他用户（包括匿名访问者）都可以对文件进行访问，这有可能造成您数据的外泄以及费用激增，请谨慎操作。
- **公共读写**：任何用户（包括匿名访问者）都可以对文件进行访问，并且向该文件写入数据。这有可能造成您数据的外泄以及费用激增，若被人恶意写入违法信息还可能会侵害您的合法权益。除特殊场景外，不建议您配置公共读写权限。

```java
public void setObjectPrivate(OSSSignResp ossSignResp) {
    String endpoint = oSSConfig.getEndpoint();
    String accessKeyId = oSSConfig.getAccessId();
    String accessKeySecret = oSSConfig.getAccessSecret();
    // 创建OSSClient实例。
    OSS ossClient = new OSSClientBuilder().build(endpoint, accessKeyId, accessKeySecret);
    // 设置文件的访问权限为私有。
    ossClient.setObjectAcl(ossSignResp.getBucket(), ossSignResp.getObjectName(), CannedAccessControlList.Private);
    // 关闭OSSClient。
    ossClient.shutdown();
}
```

