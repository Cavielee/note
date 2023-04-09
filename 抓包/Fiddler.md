# 使用 Fiddler 抓包

通过 Fidder 可以在电脑抓取手机的包：

打开`Tools -> Options`

1. Gerneral 勾选1、3、6

![image-20211206173259672](C:\Users\63190\AppData\Roaming\Typora\typora-user-images\image-20211206173259672.png)

2. HTTPS 勾选第一个，然后点击右侧选项并选择Export Root Certificate to Desktop

![image-20211206173841540](C:\Users\63190\AppData\Roaming\Typora\typora-user-images\image-20211206173841540.png)

3. 设置代理端口

默认端口为8888

![image-20211206175315057](C:\Users\63190\AppData\Roaming\Typora\typora-user-images\image-20211206175315057.png)

4. 重启软件
5. 查看电脑的ip地址，可以通过 cmd 中输入 ipconfig 查询
6. 确保手机和电脑在同一局域网
7. 设置手机 wifi 代理为手动模式，并设置为pc的ip地址和8888端口

 