# 简介

常用的构建工具中，比较常用的是 Maven 和 Ant，而 Gradle 是基于 Maven 和 Ant 的新一代构建工具。

Gradle 的主要特点如下：

1. Maven 和 Ant 使用 XML 构建，而 Gradle 是基于脚本去构建。

   Ant 对于依赖和多项目管理十分繁杂，而 Maven 通过 XML 提供更方便的管理。但由于使用 XML，随着庞大的依赖、多项目的管理，会导致 XML 配置文件十分大，臃肿。

2. Gradle 是编程语言构建的脚本，可以自定义逻辑，更加灵活。
3. 提供了大量的第三方插件库，可以轻松实现一些构建功能（编写 Task）。
4. 兼容了 Ant 和 Maven。

由于 Maven 采用的是 XML 格式的配置文件，如果项目依赖的包比较多，那么XML文件就会变得非常非常长；

1. XML文件不太灵活，如果需要在构建过程中添加一些自定义逻辑，十分麻烦；



Gradle 是继 Maven 之后的新一代构建工具，它采用基于 groovy 的 DSL 语言作为脚本，相比传统构建工具通过XML来配置而言，最直观上的感受就是脚本更加的简洁、优雅。如果你之前对Maven有所了解，那么可以很轻易的转换到 Gradle，它采用了同 Maven 一致的目录结构，可以与 Maven 一样使用 Maven 中央仓库以及各类仓库的资源，并且 Gradle 默认也内置了脚本转换命令可以方便的将 POM 转换为 gradle.build。



# 安装

由于 Gradle 运行依赖于 JVM（跨平台特性，只要有 JVM 就能运行 Gradle），因此需要预先安装 JDK/JRE。



## 下载

从 [Gralde 官方网站](https://gradle.org/releases/)下载 Gradle 的最新发行包。



## 解压

Gradle 发行包是一个 ZIP 文件。完整的发行包包括以下内容(官方发行包有 full 完整版，也有不带源码和文档的版本，可根据需求下载。



## 配置环境变量

运行 gradle 必须将 GRADLE_HOME/bin 加入到你的 PATH 环境变量中。



## 测试安装

运行如下命令来检查是否安装成功。该命令会显示当前的 JVM 版本和 Gradle 版本。

```sh
gradle -v 
```



## JVM 参数配置

Gradle 运行时的 JVM 参数可以通过 GRADLE_OPTS 或 JAVA_OPTS 来设置.这些参数将会同时生效。 JAVA_OPTS 设置的参数将会同其它 JAVA 应用共享，一个典型的例子是可以在 JAVA_OPTS 中设置代理和 GRADLE_OPTS 设置内存参数。同时这些参数也可以在 gradle 或者 gradlew 脚本文件的开头进行设置。



# gradle 项目结构

- 父项目
  - 子项目
  - 子项目
  - build.gradle
  - settings.gradle
  - gradlew



每个项目都会有一个 build.gradle，可以理解为一个 Project。只有一个根项目（等价于 Maven 的父模块），和多个子项目（等价于 Maven 的子模块）。

每个 Project 里面定义了许多 Task（命令），每一个 Task 都会有特定的逻辑处理。



## settings.gradle

配置项目基本信息，如多模块配置。

```groovy
// 根项目名
rootProject.name = 'fatherProject'
// 子项目名
include "childProject1"
include "childProject2"
```



## build.gradle

作用可以类比 Maven 的 pom.xml，每一个项目都会有一个 build.gradle 文件。里面主要由 project + task 组成。



### 依赖

```groovy
buildScript {
    dependencies {
        /** 格式如下：范围 'groupId:artifactId:version'
     	 *  version 可以不填，不填默认为最新 	
     	 */
        scope 'xxx:xxx:xxx'
    }

    repositories {
        mavenLocal()
        maven { url "https://maven.aliyun.com/repository/public" }
       	mavenCentral()
    }
}


xxxProjects {
    dependencies {
        /** 格式如下：范围 'groupId:artifactId:version'
     	 *  version 可以不填，不填默认为最新 	
     	 */
        scope 'xxx:xxx:xxx'
    }
    
    repositories {
        mavenLocal()
        maven { url "https://maven.aliyun.com/repository/public" }
       	mavenCentral()
    }
}
```

* buildScript 下的 dependencies 用于配置 gradle 自身对外部插件的依赖信息
* xxxProjects 下的 dependencies 用于配置项目本身对外部库的依赖



#### 依赖范围（scope）

* api：是 compile（已废弃） 的替代。依赖会参与编译和打包过程。

  > 例如：项目1 api A依赖，此时项目2 依赖了项目1，则项目2 可以使用项目1 的 A依赖。
  >
  > 如果项目2 此时也依赖 A依赖，则有可能导致 A依赖版本冲突（同时存在多个版本的 A依赖）。
  >
  > 版本冲突解决方案如下：
  >
  > 1. 项目2 不引入 A依赖，直接使用依赖的项目1 中的 A依赖；
  >
  > 2. 在项目2 中指定将依赖的项目 1过滤掉 A 依赖：
  >
  >    ```groovy
  >    api ("com.xxxxx.xxxxxx:xxxxx:1.0.0") {
  >        exclude group: 'org.slf4j', module: 'slf4j-api' 
  >    }
  >    ```
  >
  > 3. 强制指定某个版本
  >
  >    ```groovy
  >    configurations.all {
  >       resolutionStrategy {
  >           force 'org.slf4j:slf4j-api:1.7.25'
  >       }
  >    }
  >    ```
  >
  > 
  >
  > gradle 默认冲突策略：冲突报错
  >
  > ```groovy
  > configurations.all {
  >    resolutionStrategy {
  >        failOnVersionConflict
  >    }
  > }
  > ```
  >
  > 

  

* implementation：和 api 一样，依赖会参与编译和打包过程。（默认的scope）

  > 例如：项目1 api A依赖，此时项目2 依赖了项目1，则项目2 不可以使用项目1 的 A依赖。
  >
  > 如果项目2 此时依赖了 A依赖，则项目2 使用的是自己的 A依赖，项目1 也使用自己的 A依赖。

* compileOnly：只在编译时有效，不会参与打包。

* runtimeOnly：只在打包时有效，不会参与编译。

* testImplementation：只在单元测试代码的编译和打包测试apk时有效。



#### 仓库

用于配置依赖从哪里获取。可以配置远程 Maven 中心地址、Maven 本地仓库、ivy 仓库等。

```groovy
repositories {
    /* maven 本地仓库*/
    mavenLocal()
    /* maven 远程仓库地址*/
    maven { url "https://maven.aliyun.com/repository/public" }
    /* maven 中央仓库*/
    mavenCentral()
}
```

> 按照配置的顺序依次寻找依赖



### 依赖其他项目

同一个构建中，project 可以依赖其他project，即一个项目的 jar 包可以提供给另外一个项目使用。例如在 api 项目中，可以依赖公共项目 common：

```groovy
dependencies {
    compile project(':common')
}
```





### 项目信息配置

在根项目的 build.gradle 通过 `allprojects` 和 `subprojects` 配置项目通用信息。

简单来说就是：

* 对于所有项目通用的信息，如版本、参数、依赖等，统一配置在  `allprojects`  即可。
* 对于所有子项目通用的信息，统一配置在 `subprojects` 即可。
* 如果指定特定项目，可以通过 `project('projectname')` 来对指定项目配置。



### 扩展属性

```groovy
ext {
    springCloudVersion = "2020.0.3"
    mybatisPlusVersion = "3.4.2"
    mapstructVersion = "1.4.2.Final"
    openfeignVersion = "10.12"
    okhttpVersion = "4.9.1"
}
```

通过 `ext` 定义一些常量参数，一般用于定义依赖版本号。

通过 `key = 'val'` 的方式定义。



### 插件

```groovy
apply plugin: 'java'

// 第二种方式
plugins {
    id 'org.springframework.boot' version '2.4.7'
}
```

通过 `apply plugin 'pluginName'` 引入插件。



### jar 插件

```groovy
jar {
	baseName = 'test-jar'
	version = '0.0.1-SNAPSHOT'
}
```

在 build 时，会自动将项目打包成 jar 包放在 `build/libs` 中，可以自定义打包信息。



### 引入其他配置文件

我们可以将一些配置信息自定义在一个单独的 `xxx.gradle`，通过在 `build.gradle` 中引入该配置文件即可：

```groovy
// 引用其他gradle脚本，push.gradle 就是另外一个gradle脚本文件
apply from: './push.gradle'
```



### 拷贝

```groovy
task copyResources(type: Copy) {
	from "${projectDir}/src/main/resources"
	into "${buildDir}/classes/java/main"
}
```



### 遍历文件

```groovy
fileTree('build/resources/') { FileTree fileTree ->
	fileTree.visit { FileTreeElement element ->
		println 'the file name is:' + element.file.name
	}
}
```



### 发布

通常我们需要将打包好的 jar 包发布到仓库，此时需要通过 `uploadArchives` 任务定义发布的仓库路径在哪。

下面是通过 `maven-publish` 插件将打包好的 jar 包发布到指定仓库：

```groovy
apply plugin: 'maven-publish'

task sourceJar(type: Jar) {
    from sourceSets.main.java.srcDirs
    classifier 'sources'
}

publishing {
    // 目标仓库
    repositories {
        maven {
            // credentials {
			//	username 'username' // 仓库认证用户名
            //    password 'password' // 仓库认证密码
            // }
            url "xxx"
        }
    }   

    publications {          
        mavenJava(MavenPublication) {
            // 设置gav属性，可以不填，不填默认使用项目的属性值
            groupId 'com.cavie'
            artifactId 'gradle-test'
            version '1.0.0-RELEASE'
            from components.java // 发布jar包
            // from components.war // 发布war包
            // 打包源码
            // artifact sourceJar
        }
    }
}
```





### 创建文件

```groovy
def createDir = {
    path -> {
        File dir = new File(path)
        if (!dir.exists()) {
            dir.mkdirs
        }
    }
}

task createJavaDir() {
    def paths = ["src/main/java", "src/main/resources",
                "src/test/java", "src/test/resources"]
    paths.foreach(createDir)
}
```





### 其他

```groovy
group = 'cn.com.pcauto.autox' // groupId
version = '0.0.1-Release' // 版本

sourceCompatibility = JavaVersion.VERSION_11 // jdk 源码版本
targetCompatibility = JavaVersion.VERSION_11 // jdk 编译版本
```





## Task

task 为命令，执行后会进行一定的脚本处理。

### 任务定义

```groovy
task mytask {
    println "任务执行"
}
```



### 任务依赖

通过定义任务依赖关系，可以控制任务执行顺序

```java
task mytask {
    println "任务执行"
}
task mytask1 {
    println "任务1执行"
}
task mytask2 {
    dependsOn 'mytask1'
    println "任务2执行"
}
// mytask1依赖任务mytask
mytask1.dependsOn mytask
```



### 任务执行顺序

除了人为控制调用api和任务依赖关系可以控制任务执行顺序外，gradle 还为 task 提供了一些方法控制执行顺序：

```groovy
task mytask {
    doFirst {
        println "任务执行前执行"
    }
    println "任务执行"
    doLast {
        println "任务执行后执行"
    }
}
```



### 常用命令

* clean：删除 build 目录以及所有构建完成的文件。
* assemble：编译并打包 jar 文件，但不会执行单元测试。一些其他插件可能会增强这个任务的功能。例如，如果采用了 War 插件，这个任务便会为你的项目打出 War 包。
* build
* check：编译并测试代码。一些其他插件也可能会增强这个任务的功能。例如，如果采用了 Code-quality 插件，这个任务会额外执行 Checkstyle。

## gradlew

gradlew 即为 gradle-wrapper。

其作用为判断指定本地目录下有没有 gradle 二进制包：

* 如果没有，则从设置的下载地址下载。
* 如果有，则使用本地的二进制包。

> 缺陷在于如果本地没有 gradle 二进制包且不能联网，则此时无法运行 gradle。因此如果不能联网，一般需要在本地搭建私服、或者准备好 gradle 二进制包。



# 标准目录结构

gradle 项目约定以下目录结构为标准：

```
project root
src/main/java(测试)
src/main/resources
src/test/java(测试源码目录)
src/test/resources(测试资源目录)
src/main/webapp(web工程)
```

> Gradle 默认会从 `src/main/java` 搜寻打包源码，在 `src/test/java` 下搜寻测试源码。并且 `src/main/resources` 下的所有文件按都会被打包，所有 `src/test/resources` 下的文件 都会被添加到类路径用以执行测试。所有文件都输出到 build 下，打包的文件输出到 build/libs 下。

在使用 Idea 等工具构建 Java 项目时，默认也会创建这些目录。

如果项目目录结构和 gradle 约定的目录结构不一致，可以自定义目录结构：

```groovy
sourceSets {
    main {
        java {
            srcDir 'src/java'
        }
        resources {
            // 通过include和exclude筛选特定的资源
            srcDir 'main/resources'
            include '**/*.properties'
            include '**/*.png'


            srcDir 'src'
            include '**/Messages*.properties'
            exclude '**/*.java'
        }
    }
}
```





# gradle 执行过程

* initialization（初始化阶段）：解析整个工程中所有 Project，构建所有 Project 对应的 project 对象。
* configuration（配置阶段）：解析所有的 Project 对象中的 task，构建好所有的 task 的拓扑图。
* Execution（执行阶段）：执行具体的 task 及其依赖的 task。



# gradle 生命周期

