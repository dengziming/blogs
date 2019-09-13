---
date: 2018-04-26
title: "janusgraph源码分析1-下载编译启动"
author: "邓子明"
tags:
    - 源码
    - janusgraph
categories:
    - 源码分析
comment: true
---

#
研究了好久的 neo4j源码，现在公司要换 janusgraph，只要半途而废开始研究 janusgraph 了
`https://github.com/JanusGraph/janusgraph`和`http://janusgraph.org/`



## 一、下载编译

我直接使用github desktop打开了 janusgraph 的源码，使用IDEA打开，然后编译：
```bash
# 编译完整的
mvn -settings ~/opt/soft/apache-maven-3.5.0/conf/settings.xml -Dlicense.skip=true -DskipTests clean install
# 只编译core部分
mvn -pl janusgraph-core -am clean install -Dlicense.skip=true -DskipTests -P prod

-rf :janusgraph-test
mvn -pl janusgraph-test -am clean install -Dlicense.skip=true -DskipTests -P prod
```

我们在 `janusgraph-test` 下面编写一个例子 `FirstTest`：

```java
public class FirstTest {

    public static void main(String[] args) {

        /*
         * The example below will open a JanusGraph graph instance and load The Graph of the Gods dataset diagrammed above.
         * JanusGraphFactory provides a set of static open methods,
         * each of which takes a configuration as its argument and returns a graph instance.
         * This tutorial calls one of these open methods on a configuration
         * that uses the BerkeleyDB storage backend and the Elasticsearch index backend,
         * then loads The Graph of the Gods using the helper class GraphOfTheGodsFactory.
         * This section skips over the configuration details, but additional information about storage backends,
         * index backends, and their configuration are available in
         * Part III, “Storage Backends”, Part IV, “Index Backends”, and Chapter 13, Configuration Reference.
         */

        // Loading the Graph of the Gods Into JanusGraph
        JanusGraph graph = JanusGraphFactory
                .open("janusgraph-dist/src/assembly/cfilter/conf/janusgraph-berkeleyje-es.properties");

        GraphOfTheGodsFactory.load(graph);
        GraphTraversalSource g = graph.traversal();

        /*
         * The typical pattern for accessing data in a graph database is to first locate the entry point into the graph
         * using a graph index. That entry point is an element (or set of elements) 
         * — i.e. a vertex or edge. From the entry elements,
         * a Gremlin path description describes how to traverse to other elements in the graph via the explicit graph structure.
         * Given that there is a unique index on name property, the Saturn vertex can be retrieved.
         * The property map (i.e. the key/value pairs of Saturn) can then be examined.
         * As demonstrated, the Saturn vertex has a name of "saturn, " an age of 10000, and a type of "titan."
         * The grandchild of Saturn can be retrieved with a traversal that expresses:
         * "Who is Saturn’s grandchild?" (the inverse of "father" is "child"). The result is Hercules.
         */
        // Global Graph Indices
        Vertex saturn = g.V().has("name", "saturn").next();
        GraphTraversal<Vertex, Map<String, Object>> vertexMapGraphTraversal = g.V(saturn).valueMap();

        GraphTraversal<Vertex, Object> values = g.V(saturn).in("father").in("father").values("name");

        /*
         * The property place is also in a graph index. The property place is an edge property.
         * Therefore, JanusGraph can index edges in a graph index.
         * It is possible to query The Graph of the Gods for all events that have happened within 50 kilometers of Athens
          * (latitude:37.97 and long:23.72).
          * Then, given that information, which vertices were involved in those events.
         */
		System.out.println(g.E().has("place", geoWithin(Geoshape.circle(37.97, 23.72, 50))));
        System.out.println(g.E().has("place", geoWithin(Geoshape.circle(37.97, 23.72, 50)))
                .as("source").inV()
                .as("god2")
                .select("source").outV()
                .as("god1").select("god1", "god2")
                .by("name"));
    }

}
```
然后在"janusgraph-dist/src/assembly/cfilter/conf/janusgraph-berkeleyje-es.properties" 文件中，将注释掉的内容取消注释。

运行发现依赖挺麻烦。
首先运行报错了：
```
Exception in thread "main" java.lang.IllegalArgumentException: Could not find implementation class: org.janusgraph.diskstorage.berkeleyje.BerkeleyJEStoreManager
```
找到报错处的代码，我们发现 `janusgraph-core` 中通过反射创建一个类，但是这个类在 `janusgraph-berkeleyje` 中，而前者不依赖后者，所以找不到这个类，我们可以将后者加到前者的依赖，
但是我们发现后者依赖前者，如果加了依赖两个就相互依赖了，这是 Janus 官方设计的问题。我们只好在 FirstTest 所在的module中把两个依赖都加进来试试。
（注意，如果我们将所有的都打进一个包，这个问题就不存在了，但是在本地运行是不一样的，各自模块的编译输出文件在不同的地方。）在 `janusgraph-test` 中添加：
```xml
        <dependency>
            <groupId>org.janusgraph</groupId>
            <artifactId>janusgraph-berkeleyje</artifactId>
            <version>0.3.0-SNAPSHOT</version>
        </dependency>
```
发现 `janusgraph-berkeleyje`也依赖了 `janusgraph-test`,又相互依赖了，好麻烦。我们写写代码一定要注意这个问题。这里我的解决方法是直接把 代码放到 `janusgraph-berkeleyje` 中运行。

```
Exception in thread "main" java.lang.IllegalArgumentException: Could not find implementation class: org.janusgraph.diskstorage.es.ElasticSearchIndex
```
和上面一样，还依赖了 `janusgraph-es`,我只好吧代码复制到 `janusgraph-es` 的test代码块中运行（注意一点是test代码中），顺便在 `janusgraph-es` 中 添加上`janusgraph-berkeleyje`的依赖。
运行成功了，但是报了连接失败，是因为我本地没有启动es，我启动一下es：`elasticsearch`
然后在运行：
```java
Exception in thread "main" org.janusgraph.core.SchemaViolationException: Adding this property for key [~T$SchemaName] and value [rtname] violates a uniqueness constraint [SystemIndex#~T$SchemaName]
```
经过google查到原因： https://groups.google.com/forum/#!topic/aureliusgraphs/vZ_nTXlXj4k
```
This exception is thrown only when you already have added property key to index. So "name" is already added and next time when you run your program somewhere it is again adding "name" property key. So check if that particular code is running twice
```
然后我们可以在我们传入的配置文件找到：storage.directory=../db/berkeley  ，直接删除这个目录，再重新运行，就成功了：
```
11:20:17,051  INFO GraphDatabaseConfiguration:1285 - Set default timestamp provider MICRO
11:20:17,296  INFO GraphDatabaseConfiguration:1492 - Generated unique-instance-id=c0a815a789637-dengzimings-MacBook-Pro-local1
11:20:17,547  INFO Backend:462 - Configuring index [search]
11:20:19,279  INFO Backend:177 - Initiated backend operations thread pool of size 8
11:20:19,461  INFO KCVSLog:753 - Loaded unidentified ReadMarker start time 2018-04-26T03:20:19.408Z into org.janusgraph.diskstorage.log.kcvs.KCVSLog$MessagePuller@73cd37c0
[GraphStep(edge,[]), HasStep([place.geoWithin(BUFFER (POINT (23.72 37.97), 0.44966))])]
[GraphStep(edge,[]), HasStep([place.geoWithin(BUFFER (POINT (23.72 37.97), 0.44966))])@[source], EdgeVertexStep(IN)@[god2], SelectOneStep(last,source), EdgeVertexStep(OUT)@[god1], SelectStep(last,[god1, god2],[value(name)])]
11:20:29,578  INFO ManagementLogger:192 - Received all acknowledgements for eviction [1]
```

然后我们可以去 ../db/berkeley  目录查看，多了一些文件，这些文件的作用我们后续再分析。
然后我们取es查看：`curl -XGET 'localhost:9200/_cat/indices?v&pretty'` ，发现多了两个index:
```
yellow open   janusgraph_edges    QT-E7AV6SMWr8Cu_ywKsXg   5   1          6            0     13.7kb         13.7kb
yellow open   janusgraph_vertices gE4TSXFATnSZUWYdAf46Xg   5   1          6            0     10.9kb         10.9kb
```
还可以具体查看内容。例如名字是titan的内容：`curl -XGET 'localhost:9200/janusgraph_vertices/_search?q=name:titan&pretty'`

到现在我们第一个案例就结束了。
```java
g.E().has("place", geoWithin(Geoshape.circle(37.97, 23.72, 50)))
                .as("source").inV()
                .as("god2")
                .select("source").outV()
                .as("god1").select("god1", "god2")
                .by("name")
```
这种风格的代码实际上是groovy语言的代码，大家可以研究一下groovy语言。

注意事项：
上述第一次运行问题的原因是 `janusgraph-core`需要用到 `janusgraph-berkeleyje`的类，
但是`janusgraph-berkeleyje`是依赖 `janusgraph-core`的，所以两个相互依赖了。
janus的做法是在core中使用反射，所以编译通过了，打包到了一起就没问题了。但是本地运行没法成功。

