## SonarLint

SonarLint(Sonar) 代码质量管理



## Sonar是什么？

- Sonar是一个用于**代码质量管理**的开源平台，通过插件形式管理源代码的质量 ，可以支持包括java,C#,C/C++,PL/SQL,Cobol,JavaScrip,Groovy等等二十几种编程语言的代码质量管理与检测 
- Sonar可以从以下七个维度检测代码质量，而作为开发人员至少需要处理前5种代码质量问题 
  - **不遵循代码标准**：sonar可以通过PMD,CheckStyle,Findbugs等等代码规则检测工具规范代码编写 
  - **潜在的缺陷** ：sonar可以通过PMD,CheckStyle,Findbugs等等代码规则检测工具检测出潜在的缺陷
  - **糟糕的复杂度分布**：文件、类、方法等，如果复杂度过高将难以改变，这会使得开发人员难以理解它们 且如果没有自动化的单元测试，对于程序中的任何组件的改变都将可能导致需要全面的回归测试 
  - **重复**：显然程序中包含大量复制粘贴的代码是质量低下的，sonar可以展示源码中重复严重的地方 
  - **注释不足或者过多**：没有注释将使代码可读性变差，特别是当不可避免地出现人员变动时，程序的可读性将大幅下降 而过多的注释又会使得开发人员将精力过多地花费在阅读注释上，亦违背初衷 
  - **缺乏单元测试**：sonar可以很方便地统计并展示单元测试覆盖率 
  - **糟糕的设计**：通过sonar可以找出循环，展示包与包、类与类之间相互依赖关系，可以检测自定义的架构规则 通过sonar可以管理第三方的jar包，可以利用LCOM4检测单个任务规则的应用情况， 检测耦合。   



## SonarLint 插件安装

在 eclipse -> Help -> Eclipse Marketplace 搜索 sonar 插件

![sonar.png](https://github.com/Cavielee/note/blob/master/pics/sonar.png?raw=true)



离线下载地址：https://binaries.sonarsource.com/SonarLint-for-Eclipse/releases/

（`*.eclipse.site.*.zip`）

## 基本使用

sonarlint 有5个视图，可以通过以下方法调出

eclipse-->Window-->Show View-->other-->SonarLint

* **SonarQube Servers**

  连接sonarqube服务，点击Connect to a Sonarqube server,补充完整URL，Name，Username,password,然后点击完成。


![img](https://img-blog.csdn.net/20170415095134161?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvbHVja3lzdGFyNjg5/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)



sonarlint 仅需要配置这一步，就可以使用 sonarqube 服务的所有配置。并且，如果 sonarqube 的服务配置有修改，sonarlint 也会同步更改的。



* **sonarlint Report（可以显示当前工程，或所有工程）**

代码不规范的事项列表



![img](https://img-blog.csdn.net/20170415094512043?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvbHVja3lzdGFyNjg5/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)



点击每一项，即可跳到对应的代码上，进行相应事项的代码修改，保存，sonarlint Report会自动检测变更并刷新列表。做到了即时反馈。



* **sonarqube Rule Description**

选择sonarlint Report中的某一事项，右击，选择rule description.显示出此事项的问题所在，以及正确的代码应该如何编写等。就和我们在sonarqube页面上看到的是一样的。

![img](https://img-blog.csdn.net/20170415095757170?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvbHVja3lzdGFyNjg5/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)



* **sonarlint on-the -fly**

显示的是当前打开的文件的不规范代码描述。

![img](https://img-blog.csdn.net/20170415100216405?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvbHVja3lzdGFyNjg5/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

* **SonarLint Issue Locations**

显示的是issue的具体位置。