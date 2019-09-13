---
date: 2018-10-30
title: "tinkerpop源码解析1-简单例子debug"
author: "邓子明"
tags:
    - tinkerpop
    - 知识图谱
categories:
    - 源码
comment: true
---

tinkerpop 源码是JanusGraph 源码解析的第一步，我们需要大概有个了解。

## demo 编写

我们可以直接复制来自 tinkerpop 官方的源码：

```
public static void main(String[] args) {

    TinkerGraph graph = TinkerGraph.open();
    GraphTraversalSource g = graph.traversal();

    Vertex v = g.addV().property("name","marko").property("nam","marko a. rodriguez").next();

    GraphTraversal<Vertex, Long> name = g.V(v).properties("name").count();
    v.property(list, "name", "m. a. rodriguez");
    g.V(v).properties("name").count();
    g.V(v).properties();
    g.V(v).properties("name");
    g.V(v).properties("name").hasValue("marko");
    g.V(v).properties("name").hasValue("marko").property("acl","private"); //
    g.V(v).properties("name").hasValue("marko a. rodriguez");
    g.V(v).properties("name").hasValue("marko a. rodriguez").property("acl","public");
    g.V(v).properties("name").has("acl","public").value();
    g.V(v).properties("name").has("acl","public").drop(); //4\
    g.V(v).properties("name").has("acl","public").value();
    g.V(v).properties("name").has("acl","private").value();
    g.V(v).properties();
    g.V(v).properties().properties(); //5\
    g.V(v).properties().property("date",2014) ;//6\
    g.V(v).properties().property("creator","stephen");
    g.V(v).properties().properties();
    g.V(v).properties("name").valueMap();
    g.V(v).property("name","okram"); //7\
    g.V(v).properties("name");
    g.V(v).values("name"); //8
}
```

然后从第一行开始 打断点，debug。首先注意 TinkerGraph 是一个很简单的图数据库，超级简单。

## TinkerGraph 

TinkerGraph.open() 方法会新建一个 TinkerGraph：

```java
private TinkerGraph(final Configuration configuration) {
    this.configuration = configuration;
    vertexIdManager = selectIdManager(configuration, GREMLIN_TINKERGRAPH_VERTEX_ID_MANAGER, Vertex.class);
    edgeIdManager = selectIdManager(configuration, GREMLIN_TINKERGRAPH_EDGE_ID_MANAGER, Edge.class);
    vertexPropertyIdManager = selectIdManager(configuration, GREMLIN_TINKERGRAPH_VERTEX_PROPERTY_ID_MANAGER, VertexProperty.class);
    defaultVertexPropertyCardinality = VertexProperty.Cardinality.valueOf(
            configuration.getString(GREMLIN_TINKERGRAPH_DEFAULT_VERTEX_PROPERTY_CARDINALITY, VertexProperty.Cardinality.single.name()));

    graphLocation = configuration.getString(GREMLIN_TINKERGRAPH_GRAPH_LOCATION, null);
    graphFormat = configuration.getString(GREMLIN_TINKERGRAPH_GRAPH_FORMAT, null);

    if ((graphLocation != null && null == graphFormat) || (null == graphLocation && graphFormat != null))
        throw new IllegalStateException(String.format("The %s and %s must both be specified if either is present",
                GREMLIN_TINKERGRAPH_GRAPH_LOCATION, GREMLIN_TINKERGRAPH_GRAPH_FORMAT));

    if (graphLocation != null) loadGraph();
}
```
这里的 configuration 类似一个map。然后有几个 IDManager 和其他变量赋值。

然后是 GraphTraversalSource g = graph.traversal(); 这一句仅仅是 return new GraphTraversalSource(this);
```java
public GraphTraversalSource(final Graph graph) {
    this(graph, TraversalStrategies.GlobalCache.getStrategies(graph.getClass()));
}
```

这时候我们取看一下几个特殊的类： Traversal<S,E> Traversal.Admin<S, E> 。 他们两个都是接口，而且 Admin 是继承自 Traversal。
Traversal 代表遍历，主要方法包括 next， asNext，iterator toList 等，可以看成一个迭代器，而 Admin 的主要方法和他没有什么关系。

GraphTraversal<S, E> extends Traversal<S, E> , 新增加了很多和 gremlin 相关的方法。这属于 tinkerpop 的内容

然后是 addVertex 

```java
public Vertex addVertex(final Object... keyValues) {
    ElementHelper.legalPropertyKeyValueArray(keyValues);
    Object idValue = vertexIdManager.convert(ElementHelper.getIdValue(keyValues).orElse(null));
    final String label = ElementHelper.getLabelValue(keyValues).orElse(Vertex.DEFAULT_LABEL);

    if (null != idValue) {
        if (this.vertices.containsKey(idValue))
            throw Exceptions.vertexWithIdAlreadyExists(idValue);
    } else {
        idValue = vertexIdManager.getNextId(this);
    }

    final Vertex vertex = new TinkerVertex(idValue, label, this);
    this.vertices.put(vertex.id(), vertex);

    ElementHelper.attachProperties(vertex, VertexProperty.Cardinality.list, keyValues);
    return vertex;
}
```
我们可以看出源码十分简单，而且 property 等方法就更简单了，所以 TinkerGraph 可以用来进行后续的分析。这样有关图操作部分就不会有理解难度。