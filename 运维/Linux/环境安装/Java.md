**JDK 是开发Java程序必须安装的软件，我们查看一下 yum 源里面的 JDK：**

```sh
yum list java*
```



**选择适合本机的JDK，并安装：**

```sh
yum install java-1.8.0-openjdk* -y
```



**安装完成后，查看是否安装成功：**

```sh
java -version
```

