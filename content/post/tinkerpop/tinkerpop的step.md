---
date: 2018-07-27
title: "tinkerpop 的 step"
author: "邓子明"
tags:
    - tinkerpop
    - 源码
categories:
    - 源码分析
comment: true

---

## 一、简单调试

api地址： http://tinkerpop.apache.org/javadocs/current/full/

第一步：
```java
JanusGraph graph = JanusGraphFactory.open("janusgraph-dist/src/assembly/cfilter/conf/janusgraph-berkeleyje-es.properties");

GraphTraversalSource g = graph.traversal();

g.V().has("name", "saturn").path();

List<Path> paths = path.toList();

```


一步一步看整个调用过程：

进入: fill:179, Traversal (org.apache.tinkerpop.gremlin.process.traversal)

fill 方法的代码：
```java
final Step<?, E> endStep = this.asAdmin().getEndStep();
while (true) {
    final Traverser<E> traverser = endStep.next();
    TraversalHelper.addToCollection(collection, traverser.get(), traverser.bulk());
```

asAdmin 得到 endStep，有点类似 spark 的 stage 拆分后得到 shuffleMapTask。然后调用 endStep.next() 得到 traverser。

这里的代码我们前面已经熟悉过了，再看一下。进入： next:128, AbstractStep (org.apache.tinkerpop.gremlin.process.traversal.step.util)
```java
final Traverser.Admin<E> traverser = this.processNextStart();
if (null != traverser.get() && 0 != traverser.bulk())
    return this.prepareTraversalForNextStep(traverser);
```

进入 processNextStart:118, PathStep (org.apache.tinkerpop.gremlin.process.traversal.step.map)
`return PathProcessor.processTraverserPathLabels(super.processNextStart(), this.keepLabels);` 
可以看出调用了父类的 processNextStart 方法，

进入 processNextStart:36, MapStep (org.apache.tinkerpop.gremlin.process.traversal.step.map)

由于是 mapStep，所以类似 spark 的 mapPartitionsRdd ，逻辑就是得到前面的 rdd，然后执行 map 方法的逻辑。
所以这里 mapStep 也是一样，得到  starts 的 next，然后调用map。
```java
final Traverser.Admin<S> traverser = this.starts.next();
return traverser.split(this.map(traverser), this);
```

进入 next:50, ExpandableStepIterator (org.apache.tinkerpop.gremlin.process.traversal.step.util)，我们说过这就是对 hostStep 的一个封装。主要就是
```java
if (this.hostStep.getPreviousStep().hasNext())
   return this.hostStep.getPreviousStep().next();
```
这个 hostStep 就是上面的 mapStep。这里有 getPreviousStep 然后 next。

然后又进入到了 processNextStart:142, GraphStep (org.apache.tinkerpop.gremlin.process.traversal.step.map)，
这里的 iteratorSupplier 变量其实是在 GraphStep 或者他的子类中赋值的，所以 get 方法得到的就是：

```java
public JanusGraphStep(final GraphStep<S, E> originalStep) {
    super(originalStep.getTraversal(), originalStep.getReturnClass(), originalStep.isStartStep(), originalStep.getIds());
    originalStep.getLabels().forEach(this::addLabel);
    this.setIteratorSupplier(() -> {
        if (this.ids == null) {
            return Collections.emptyIterator();
        }
        else if (this.ids.length > 0) {
            final Graph graph = (Graph)traversal.asAdmin().getGraph().get();
            return iteratorList((Iterator)graph.vertices(this.ids));
        }
        if (hasLocalContainers.isEmpty()) {
            hasLocalContainers.put(new ArrayList<>(), new QueryInfo(new ArrayList<>(), 0, BaseQuery.NO_LIMIT));
        }
        final JanusGraphTransaction tx = JanusGraphTraversalUtil.getTx(traversal);
        final GraphCentricQuery globalQuery = buildGlobalGraphCentricQuery(tx);

        final Multimap<Integer, GraphCentricQuery> queries = ArrayListMultimap.create();
        if (globalQuery != null && !globalQuery.getSubQuery(0).getBackendQuery().isEmpty()) {
            queries.put(0, globalQuery);
        } else {
            hasLocalContainers.entrySet().forEach(c -> queries.put(c.getValue().getLowLimit(), buildGraphCentricQuery(tx, c)));
        }

        final GraphCentricQueryBuilder builder = (GraphCentricQueryBuilder) tx.query();
        final List<Iterator<E>> responses = new ArrayList<>();
        queries.entries().forEach(q ->  executeGraphCentryQuery(builder, responses, q));

        return new MultiDistinctOrderedIterator<E>(lowLimit, highLimit, responses, orders);
    });
}
```

从这段代码，结合前面我们分析过的 GraphStep ，我们看出和图相关的 GraphStep 主要就是有一个 iteratorSupplier。因为这个step 就是为了从图拿数据。

我们再看看别的 Step。

## 简单 Step 查看

其实我们查看 Step 主要就是了解 processNextStart 的行为，接下来先看几个简单的。

简单的 step 一般只处理一个逻辑，类似 spark 中的 map flatMap filter 等方法。

### MapStep

MapStep 是抽象类，表示这个Step有很多实现，需要自己继承。processNextStart 方法就是调用 starts 的next 返回一个Traverser，然后调用 map(返回一个Traverser);

```
protected Traverser.Admin<E> processNextStart() {
    final Traverser.Admin<S> traverser = this.starts.next();
    return traverser.split(this.map(traverser), this);
}
```
MapStep 有很多的实现类，例如：PropertyKeyStep LabelStep PropertyValueStep PathStep MathStep EdgeOtherVertexStep 等，他们的 map 方法实现很简单。

### FilterStep 

和 MapStep 类似，它的子类有 WhereStep HasStep NotStep CoinStep IsStep 等。

### FlatMapStep 

和 MapStep 类似，它的子类有 EdgeVertexStep VertexStep PropertiesStep 等。

### AggregateStep

听名字是聚合的意思，应该是多个结果合并。内部有个 TraverserSet<S> barrier 代表所有待合并的 Traverser。

### GroupStep

我们可以写一段代码测试一下：g.V().group().by(T.label).next()

1. this.asAdmin().addStep(new GroupStep<>(this.asAdmin()));
2. this.asAdmin().getEndStep()).modulateBy(token);
2. 1. new TokenTraversal(token)
2. 2. GroupStep.modulateBy(final Traversal.Admin<?, ?> kvTraversal)
3. 1. this.seed = this.reducingBiOperator.apply(this.seed, this.projectTraverser(this.starts.next()));
3. 2. GroupStep.doFinalReduction((Map<K, Object>) object, this.valueTraversal);


