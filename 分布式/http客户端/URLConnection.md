URLConnectionHelp 是使用 JDK 原生提供的，无需依赖其他jar包。

> 建议使用 HttpURLConnection

```java
package com.example.httpclient;

import java.io.BufferedReader;
import java.io.InputStream;
import java.io.InputStreamReader;
import java.net.URL;
import java.net.URLConnection;

/**
 * @author CavieLee
 * @since 2022/03/21
 */
public class URLConnectionHelper {
    public static String sendRequest(String urlParam) {

        URLConnection con = null;

        BufferedReader buffer = null;
        StringBuffer resultBuffer = null;

        try {
            URL url = new URL(urlParam);
            con = url.openConnection();

            //设置请求需要返回的数据类型和字符集类型
            con.setRequestProperty("Content-Type", "application/json;charset=utf-8");
            //允许写出
            con.setDoOutput(true);
            //允许读入
            con.setDoInput(true);
            //不使用缓存
            con.setUseCaches(false);
            //得到响应流
            InputStream inputStream = con.getInputStream();
            //将响应流转换成字符串
            resultBuffer = new StringBuffer();
            String line;
            buffer = new BufferedReader(new InputStreamReader(inputStream, "utf-8"));
            while ((line = buffer.readLine()) != null) {
                resultBuffer.append(line);
            }
            return resultBuffer.toString();

        } catch (Exception e) {
            e.printStackTrace();
        }

        return "";
    }

    public static void main(String[] args) {
        String url = "http://localhost:8088/ad/adPosition/list";
        System.out.println(sendRequest(url));
    }
}

```

