1. 到以下网址下载相应版本的exe文件

https://repo1.maven.org/maven2/com/google/protobuf/protoc/

2. 把其路径名放在windows环境变量下的 path 下

3. 通过该命令将定义好的 `.proto` 文件编译为java文件

   ```
   protoc -I=$SRC_DIR --java_out=$DST_DIR $SRC_DIR/xxx.proto
   ```

* -I 后面是 proto 文件所在的目录， 
* --java_out 后面是生成java文件存放地址 
* 最后一行是 proto 文件的名称，可以写绝对地址，也可以直接写 proto 文件名称



注意：建议在项目所在目录轮行该命令，且定义 java_out 为 `src/main/java`（因为.proto 文件中定义的packet 所在位置）。如：

```
protoc -I=src/main/resources/protoc --java_out=src/main/java time.proto
```

