---
date: 2018-03-22
title: "neo4j源码分析1-编译打包启动"
author: "邓子明"
tags:
    - 源码
    - neo4j
    - 大数据
categories:
    - 源码分析
comment: true
---

## 1.打包

### 1.打包community

进入community,neo4j-graphdb-api，
注释掉common的：
```xml
<plugin>
  <groupId>org.revapi</groupId>
  <artifactId>revapi-maven-plugin</artifactId>
</plugin>
```
里面好像涉及到了版本检查，如果某个类的最新发布版本已经没有这个方法，打包会失败，反正对打包有影响，不删除可能会失败。

还可能要在主项目的pom里面注释掉：`maven-checkstyle-plugin`，代码风格检查可能会通不过。
然后用maven命令：

```bash
mvn -settings ~/opt/soft/apache-maven-3.5.0/conf/settings.xml -Dlicense.skip=true -DskipTests package install
```

### 2.打包企业版

进入enterprise,ha目录
进入management,注释掉 <groupId>org.revapi</groupId>
还有其他问题，比如java文件没有license，这里不一一列举。
```bash
mvn -settings ~/opt/soft/apache-maven-3.5.0/conf/settings.xml -Dlicense.skip=true -DskipTests package install
```


### 3. 打包完整的tar包
进入项目路径

```bash
mvn clean install -Dmaven.test.skip=true
```

要注意两个参数的异同点：

```

-DskipTests，不执行测试用例，但编译测试用例类生成相应的class文件至target/test-classes下。

-Dmaven.test.skip=true，不执行测试用例，也不编译测试用例类。
```

打包的输出文件：packaging/standalone/target/neo4j-community-3.4.0-SNAPSHOT-unix.tar.gz，这个就是我们的neo4j包。解压后，放到一个目录。一方面你可以选择执行 bin/neo4j start 启动neo4j，我们要分析源码，自然会是在本地启动。

## 二、运行

### 1.启动

我们在IDEA中，找到入口类：org.neo4j.server.CommunityEntryPoint，点击运行，然后会报错，我们需要添加运行参数：

-server --home-dir=~/neo4j-community-3.2.6 --config-dir=~/neo4j-community-3.2.6/conf

这里的参数是刚刚解压的neo4j目录和配置文件。然后运行成功，访问 http://localhost:7474/browser/，会发现有问题。
通过调试前端的js代码，我们发现版本有问题，这里我们稍作修改，找到 org.neo4j.kernel.internal.Version。最后的代码注释掉，换成我们的版本，也就是将Version.class.getPackage().getImplementationVersion() 换成 3.4，然后就可以运行成功了。
打开7474端口，写cypher语言，查看。


### 2.打断点调试

既然是源码分析，我们的办法就是先看，然后打断点调试，查看调用栈，但是由于是多线程，其实还是很有难度的，容易跟丢，后续我们慢慢来吧。



### 3.代码结构查看
看源码之前我们先大概过一下代码结构。我们主要看 community 模块的结构，里面有很多子模块。

我们可以大概根据名字猜测 ：io模块是用来处理读写数据的，kernel模块是我们需要着重查看的。bolt是处理bolt连接的，server是整个项目启动的。codegen是动态生成代码的。我们要从内核部分开始看。

### 4.架构了解

The node records contain only a pointer to their first property and their first relationship (in what is oftentermed the _relationship chain). From here, we can follow the (doubly) linked-list of relationships until we find the one we’re interested in, the LIKES relationship from Node 1 to Node 2 in this case. Once we’ve found the relationship record of interest, we can simply read its properties if there are any via the same singly-linked list structure as node properties, or we can examine the node records that it relates via its start node and end node IDs. These IDs, multiplied by the node record size, of course give the immediate offset of both nodes in the node store file.

这段话来自<Graph Databases>(作者：IanRobinson) 一书。描述了neo4j的存储方式。详情可以查阅其他资料。

### 5.源码查看
参考下一篇
