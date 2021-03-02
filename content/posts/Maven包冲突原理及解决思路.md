---
title: "Maven包冲突原理及解决思路"
date: 2021-03-02T10:32:29+08:00
draft: false
---

# Maven包冲突原理及解决思路
## Maven包冲突原理
当程序运行抛出如下异常时：

* AbstractMethodError
* NoClassDefFoundError
* NoSuchMethodError
* LinkageError
* ClassNotFoundException

十有八九是发生了包冲突

## 为什么会产生包冲突?
当我们在pom.xml引入了A库，A库又可能会依赖B库，Maven在解析pom时会依次将A、B全部引入进来，就产生了依赖传递。  
那么假设：  
A -> B -> C -> D(version1.0)  
E -> F -> D(version2.0)  
当pom引入A、E两个依赖后，根据依赖传递的原则名为D的jar包两个版本都会被引入  
而实际运行时，由于全限定类名是类的唯一标识，Maven不会将两个版本的D库拼到classpath后面(避免出现classpath hell),MAVEN会根据就近原则(最短路径优先)匹配到D(version2.0)拼到classpath后面运行程序。这时如果代码中用的都是D包1.0版本的方法就可能出现问题。

## Maven包冲突解决思路及方案
这里直接举个栗子:
先看控制台报错信息
![](/images/err.jpg)
第三方库中的方法报错，分析错误原因为：包冲突
## 查看项目依赖的方法
第一种：在idea中点击右边的maven窗口(实际项目中依赖树可能很长，难用肉眼排查)
![](/images/dependency.jpg)
第二种:在终端里执行mvn命令，重定向到文件中(注意：显示的依赖树是解决冲突之后的！！！)  
`$ mvn dependency:tree > yourFileName.txt`
第三种：在idea plugins中下载Maven Helper插件(推荐)
如下图所示：spring-web4.3.6根据maven包管理的就近原则被忽略掉了
![](/images/helper.jpg)
最后回到报错信息，在org.springframework.http.converter.json.MappingJacksonValue中没有找到getJsonpFunction()方法  
最后我们需要去看一下两个版本的源码，确认是哪里出了问题  
第一步，在maven仓库中找到spring-web
![](/images/maven.jpg)
第二步，点击HomePage的链接跳转到github,定位到org.springframework.http.converter.json.MappingJacksonValue(报错的类)中切换版本，查找问题
可以看到我们在5.1.8的版本中找不到getJsonpFunction()方法
![](/images/5.jpg)
而在4.3.6的版本中可以找到
![](/images/4.jpg)
最后回到pom文件，把直接引用的5.1.8版本注释掉，问题解决
![](/images/pom2.jpg)
以我的代码为例，是因为包冲突导致classpath中添加了错误的spring-web版本导致没有找到相应的方法。实际开发过程中，遇到的问题可能更复杂，依赖树也很繁杂，只要能够梳理清楚依赖的关系，知道代码中实际发生了什么，最后定位到源码的不同版本，确定问题，应该都是可以解决的。

第一次写博客，如果有写的不对、不容易理解的地方，欢迎指正!