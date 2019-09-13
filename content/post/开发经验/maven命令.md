---
date: 2018-03-22
title: "常用maven命令"
author: "邓子明"
tags:
    - maven
categories:
    - 开发经验
comment: true
---

## 



## 一、

### 1.插件


版本兼容检查
```xml
<plugin>
  <groupId>org.revapi</groupId>
  <artifactId>revapi-maven-plugin</artifactId>
</plugin>
```

代码风格检查
```xml
maven-checkstyle-plugin
```

### 

### 2.常见异常

1. 
cached in local repository ...
这个原因可能是上一次更新失败，会有一个update文件放在本地，去repository对应目录删除即可。

也有可能是打的包是 war 包不是jar包，复制一份，改名为 .jar 即可。

2. 
can't find jar in alimaven， 这个原因是仓库可能没这个jar，可是换一个仓库。
例如换成开源中国的仓库，但是可能又有新的错误，最好是使用 -rf 从某一个重新开始打包 `mvn clean install -DskipTests=true -rf :janusgraph-es` 

3. 
java return cannot find symbol com.sun.*
可以换成 jdk1.8.

4. 
Missing tools.jar at: /Library/Java/JavaVirtualMachines/jdk1.8.0_121.jdk/Contents/Home/Classes/classes.jar. Expression: file.exists()

原因，maven 的配置文件的 profiles 中有个 profile：
```
<profile>
  <id>osx-jdk</id>
  <activation>
    <file>
      <exists>${java.home}/../Classes/classes.jar</exists>
    </file>
  </activation>
  <dependencies>
    <dependency>
      <groupId>jdk.tools</groupId>
      <artifactId>jdk.tools</artifactId>
      <scope>system</scope>
      <version>1.6</version>
      <systemPath>${java.home}/../Classes/classes.jar</systemPath>
    </dependency>
  </dependencies>
</profile>
```

activation 代表使用条件，${java.home} 代表 JRE 目录，如果这个文件存在就是用 osx-jdk。现在是因为这个文件不存在，所以只要让这个文件存在即可。

解决办法：https://stackoverflow.com/questions/23971229/maven-install-hadoop-from-source-looking-in-the-wrong-path-for-tools-jar

cd /Library/Java/JavaVirtualMachines/jdk1.8.0_121.jdk/Contents/Home/
sudo mkdir Classes
cd Classes/
sudo ln -s ../jre/lib/rt.jar classes.jar


5. 
neo4j-3.4/community/dbms/src/main/java/org/neo4j/dbms/diagnostics/jmx/LocalVirtualMachine.java:[23,28] package com.sun.tools.attach does not exist

这个和上面的错误很类似，上面的错误是因为我们通过在 jre 目录下面做了一个软连接，把 rt.jar 连接到了 classes.jar，classes.jar 包中没有 com.sun.tools.attach，而它需要的是 tools.jar，

解决办法：

cd /Library/Java/JavaVirtualMachines/jdk1.8.0_121.jdk/Contents/Home/
sudo rm -rf Classes/classes.jar
sudo ln -s lib/tools.jar Classes/classes.jar

6. 

not found: type InputPosition
not found: value Eagerly

解决方案：
一般是 scala 编译器没配置好。重新设置。
有时候 IDEA 运行没问题，但是打包报这个问题，解决方案？
目前不知道，重新 import 后，找个类运行一下，然后再打包就成功了。


mvn -settings ~/opt/soft/apache-maven-3.5.0/conf/settings.xml -Dlicense.skip=true -DskipTests package install -rf :neo4j-cypher-util-3.4


7. 
inspect a maven model for resolution

Maven -> Reimport 

8. 
/Users/dengziming/opt/sourcecode/neo4j-3.4/community/cypher/interpreted-runtime/src/test/scala/org/neo4j/cypher/internal/runtime/interpreted/commands/ComparablePredicateTest.scala:53: warning: a type was inferred to be `Any`; this may indicate a programming error.
[ERROR]     case v: Number if v.doubleValue().isNaN => Seq(v.doubleValue(), v.floatValue(), v)
[ERROR]          ^

这个问题好像没造成停止。

9. 应用A直接应用B，应用B依赖二方包C1、C2、C3，应用A传递依赖C1、C2、C3。现应用B升级版本，应用更新B依赖包后发现可正常引入依赖B，但传递依赖的C1、C2、C3不能引入。 
　　
https://blog.csdn.net/xktxoo/article/details/78005817

10. 打包不包含 scala 编译文件，pom 的 build 中缺少打包scala的插件。

11. java 调用 scala 代码不报错，打包报错。
参考 http://xflin.blogspot.com/2013/08/mixed-scala-and-java-in-maven-project.html ，混合编译

### 3.常用命令

mvn -pl hadoop-yarn-project -am clean install -D maven.test.skip=true -P prod



