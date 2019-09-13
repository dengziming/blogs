---
date: 2018-11-22
title: "tinkerpop源码解析2-查询流程debug"
author: "邓子明"
tags:
    - tinkerpop
    - 初学
categories:
    - tinkerpop
comment: true

---

## 一、关键类

api地址： http://tinkerpop.apache.org/javadocs/current/full/

### GraphTraversalSource
构造方法：
GraphTraversalSource(Graph graph) 
GraphTraversalSource(Graph graph, TraversalStrategies traversalStrategies) 

graph.traversal() 方法返回一个 GraphTraversalSource ，我们挺好奇怎么转化为计算逻辑的，可能类似spark一样，记录操作，然后计算。但是不同的数据库怎么切换逻辑呢？

### GraphTraversal

代表了遍历，在graph中的一条路径。

### Traverser

代表 GraphTraversal 中的对象遍历过程的当前状态

## 二、简单调试

我们使用自带的一个库进行调试：

```java
TinkerGraph graph = TinkerFactory.createModern();

GraphTraversalSource g = graph.traversal();
Object next = g.V(1).outE("knows").inV().values("name").next();

System.out.println(next.toString());


```

然后我们的断点设置在 TinkerGraph 的增删改查方法上。

第一次是 `org.apache.tinkerpop.gremlin.tinkergraph.structure.TinkerGraph.addVertex(TinkerGraph.java:161)`，这个方法的内容比较简单，得到 id label ，新建 TinkerVertex。

然后是`TinkerGraph.vertices`,完整调用信息：
```
org.apache.tinkerpop.gremlin.tinkergraph.structure.TinkerGraph.vertices(TinkerGraph.java:245)
org.apache.tinkerpop.gremlin.tinkergraph.process.traversal.step.sideEffect.TinkerGraphStep.vertices(TinkerGraphStep.java:85)
org.apache.tinkerpop.gremlin.tinkergraph.process.traversal.step.sideEffect.TinkerGraphStep.lambda$new$0(TinkerGraphStep.java:59)
org.apache.tinkerpop.gremlin.tinkergraph.process.traversal.step.sideEffect.TinkerGraphStep$$Lambda$23.846254484.get(Unknown Source:-1)
org.apache.tinkerpop.gremlin.process.traversal.step.map.GraphStep.processNextStart(GraphStep.java:142)
org.apache.tinkerpop.gremlin.process.traversal.step.util.AbstractStep.hasNext(AbstractStep.java:143)
org.apache.tinkerpop.gremlin.process.traversal.step.util.ExpandableStepIterator.next(ExpandableStepIterator.java:50)
org.apache.tinkerpop.gremlin.process.traversal.step.map.FlatMapStep.processNextStart(FlatMapStep.java:48)
org.apache.tinkerpop.gremlin.process.traversal.step.util.AbstractStep.hasNext(AbstractStep.java:143)
org.apache.tinkerpop.gremlin.process.traversal.step.util.ExpandableStepIterator.next(ExpandableStepIterator.java:50)
org.apache.tinkerpop.gremlin.process.traversal.step.map.FlatMapStep.processNextStart(FlatMapStep.java:48)
org.apache.tinkerpop.gremlin.process.traversal.step.util.AbstractStep.next(AbstractStep.java:128)
org.apache.tinkerpop.gremlin.process.traversal.step.util.AbstractStep.next(AbstractStep.java:38)
org.apache.tinkerpop.gremlin.process.traversal.util.DefaultTraversal.next(DefaultTraversal.java:200)
我们的代码：Object next = g.V(1).outE("knows").inV().values("name").next();
```

一共出现了 TinkerGraphStep，GraphStep，AbstractStep，FlatMapStep 四类 Step，最后一个 TinkerGraphStep 是怎么出现的？按理说都是tinkerpop 的实现，其实是  TinkerGraphStepStrategy 的缘故。

接下来我们吧 `Object next = g.V(1).outE("knows").inV().values("name").next();` 拆开分析：

```java
GraphTraversal<Vertex, Vertex> v = g.V(1);
GraphTraversal<Vertex, Edge> knows = v.outE("knows");
GraphTraversal<Vertex, Vertex> inV = knows.inV();
GraphTraversal<Vertex, Object> name = inV.values("name");
Object next = name.next();
```

每一步我们都断点，然后查看当前 GraphTraversal 的属性，状态等。

1. 注册  traversalStrategies ：

```
traversalStrategies = {ArrayList@1426}  size = 14
 0 = {ConnectiveStrategy@1428} "ConnectiveStrategy"
 1 = {IncidentToAdjacentStrategy@1429} "IncidentToAdjacentStrategy"
 2 = {MatchPredicateStrategy@1430} "MatchPredicateStrategy"
 3 = {FilterRankingStrategy@1431} "FilterRankingStrategy"
 4 = {InlineFilterStrategy@1432} "InlineFilterStrategy"
 5 = {AdjacentToIncidentStrategy@1433} "AdjacentToIncidentStrategy"
 6 = {RepeatUnrollStrategy@1434} "RepeatUnrollStrategy"
 7 = {PathRetractionStrategy@1435} "PathRetractionStrategy"
 8 = {CountStrategy@1436} "CountStrategy"
 9 = {LazyBarrierStrategy@1437} "LazyBarrierStrategy"
 10 = {TinkerGraphCountStrategy@1438} "TinkerGraphCountStrategy"
 11 = {TinkerGraphStepStrategy@1439} "TinkerGraphStepStrategy"
 12 = {ProfileStrategy@1440} "ProfileStrategy"
 13 = {StandardVerificationStrategy@1441} "StandardVerificationStrategy"
```

2. g.V(id)

```java
public GraphTraversal<Vertex, Vertex> V(final Object... vertexIds) {
    final GraphTraversalSource clone = this.clone(); // 克隆 对象
    clone.bytecode.addStep(GraphTraversal.Symbols.V, vertexIds); // 记录
    final GraphTraversal.Admin<Vertex, Vertex> traversal = new DefaultGraphTraversal<>(clone);
    {// 这里逻辑很简单，直接赋值
        this.graph = graph;
        this.strategies = traversalStrategies;
        this.bytecode = bytecode;
    }
    return traversal.addStep(new GraphStep<>(traversal, Vertex.class, true, vertexIds));
    {
    // 1. new GraphStep<>(traversal, Vertex.class, true, vertexIds)
        {
        super(traversal);// AbstractStep
        // 这一步要注意，这里的 iteratorSupplier 使用了 getGraph。如果graph 是 JanusGraph，那么就会查询 JanusGraph 的相关方法。
        this.iteratorSupplier = () -> (Iterator<E>) (Vertex.class.isAssignableFrom(this.returnClass) ?
             this.getTraversal().getGraph().get().vertices(this.ids) :
             this.getTraversal().getGraph().get().edges(this.ids));
        }
    
    // 2. traversal.addStep
    {
    return (GraphTraversal.Admin<S, E2>) Traversal.Admin.super.addStep((Step) step);
    {
        public <S2, E2> Traversal.Admin<S2, E2> addStep(final int index, final Step<?, ?> step) throws IllegalStateException {
        	if (this.locked) throw Exceptions.traversalIsLocked();
        	step.setId(this.stepPosition.nextXId());
        	this.steps.add(index, step); // 添加到集合
        	final Step previousStep = this.steps.size() > 0 && index != 0 ? steps.get(index - 1) : null;
        	final Step nextStep = this.steps.size() > index + 1 ? steps.get(index + 1) : null;
        	// 设置前后依赖
        	step.setPreviousStep(null != previousStep ? previousStep : EmptyStep.instance());
        	step.setNextStep(null != nextStep ? nextStep : EmptyStep.instance());
        	if (null != previousStep) previousStep.setNextStep(step);
        	if (null != nextStep) nextStep.setPreviousStep(step);
        	step.setTraversal(this);
        	return (Traversal.Admin<S2, E2>) this;
    	}
    }
    }
    }
}
```

3. v.outE("knows");

```java
public default GraphTraversal<S, Edge> outE(final String... edgeLabels) {
    this.asAdmin().getBytecode().addStep(Symbols.outE, edgeLabels); // 记录
    return this.asAdmin().addStep(new VertexStep<>(this.asAdmin(), Edge.class, Direction.OUT, edgeLabels));
    {
    // 1. new VertexStep<>(this.asAdmin(), Edge.class, Direction.OUT, edgeLabels)
        {
        super(traversal);// FlatMapStep -> AbstractStep
        this.direction = direction;
        this.edgeLabels = edgeLabels;
        this.returnClass = returnClass;
        }
    // 2. addStep
    // 和上面一样
    }
}
```

4. knows.inV()

```java
public default GraphTraversal<S, Vertex> inV() {
    this.asAdmin().getBytecode().addStep(Symbols.inV); 
    return this.asAdmin().addStep(new EdgeVertexStep(this.asAdmin(), Direction.IN));
    {
    // 1. new EdgeVertexStep(this.asAdmin(), Direction.IN)
    {
        public EdgeVertexStep(final Traversal.Admin traversal, final Direction direction) {
        super(traversal);
        this.direction = direction;
        }
    }
    // 2. 和上面一样
    }
}
```

5. inV.values("name");

```java
public default <E2> GraphTraversal<S, E2> values(final String... propertyKeys) {
    this.asAdmin().getBytecode().addStep(Symbols.values, propertyKeys);
    return this.asAdmin().addStep(new PropertiesStep<>(this.asAdmin(), PropertyType.VALUE, propertyKeys));
    {
    // 1. new PropertiesStep<>(this.asAdmin(), PropertyType.VALUE, propertyKeys)
    {
        public PropertiesStep(final Traversal.Admin traversal, final PropertyType propertyType, final String... propertyKeys) {
        super(traversal);
        this.returnType = propertyType;
        this.propertyKeys = propertyKeys;
    	}
    }
    }
}
```

6. name.next();

next 方法类似spark中的 action，

```java

public E next() {
    try {
        if (!this.locked) this.applyStrategies();
        if (this.lastTraverser.bulk() == 0L) // 这个判断的意义是什么。
            this.lastTraverser = this.finalEndStep.next();
        this.lastTraverser.setBulk(this.lastTraverser.bulk() - 1L);
        return this.lastTraverser.get();
    } catch (final FastNoSuchElementException e) {
        throw this.parent instanceof EmptyStep ? new NoSuchElementException() : e;
    }
}
```

可以看出这里调用了 this.finalEndStep.next()，有点像spark，将逻辑转到 Step 中。

这个 next 方法触发了查询等操作，所以逻辑会比其他的复杂，我们可以看看几个关键步骤：

```java

this.applyStrategies();
this.lastTraverser = this.finalEndStep.next();
this.lastTraverser.setBulk(this.lastTraverser.bulk() - 1L);
```

涉及到几个属性：

```java
private Traverser.Admin<E> lastTraverser = EmptyTraverser.instance();
private Step<?, E> finalEndStep = EmptyStep.instance();  
protected List<Step> steps = new ArrayList<>(); 
```

在 applyStrategies 方法中有 `this.finalEndStep = this.getEndStep();` 我们可以知道一个 Traversal 有很多steps，前后都有依赖关系，有一个 finalEndStep。
然后是 `this.lastTraverser = this.finalEndStep.next();` 可以看出 step 的 next 方法很重要，返回一个 Traverser ,Traverser 能干嘛我们现在还不知道。
但是我们现在看出 Traverser 有 bulk 属性，如果 bulk >0 证明当前已经执行过了，每次调用 hasNext 或者 next 的时候就直接返回数据，否则需要查询。
而查询的逻辑很明显就在 `this.lastTraverser = this.finalEndStep.next();`

7. AbstractStep.next()

Step 继承自Iterator，Step.next() 返回一个 Traverser，中间涉及了查询的操作。next 实现 在 AbstractStep 中，还有两个类实现了重写。重写也会调用父类。

```java
public Traverser.Admin<E> next() {
    if (null != this.nextEnd) { // nextEnd 应该是缓存，如果不为空直接返回，否则查询。
        try {
            return this.prepareTraversalForNextStep(this.nextEnd);
        } finally {
            this.nextEnd = null;
        }
    } else {
        while (true) {
            if (Thread.interrupted()) throw new TraversalInterruptedException();
            final Traverser.Admin<E> traverser = this.processNextStart();
            if (null != traverser.get() && 0 != traverser.bulk())
                return this.prepareTraversalForNextStep(traverser);
        }
    }
}
```

然后调用 processNextStart()，这就是真正的逻辑发生的地方。processNextStart 在 AbstractStep 中是一个抽象方法，需要底层不同的实现。就拿我们这里的几个例子来说：

```
0 = {GraphStep@1408} "GraphStep(vertex,[1])"    GraphStep 是最原始的step，它的next 方法是从数据库拿数据。所以 会保存一个 iterator，还有个 iteratorSupplier。
1 = {VertexStep@1409} "VertexStep(OUT,[knows],edge)" 继承自 FlatMapStep，内部保存了一个 ExpandableStepIterator starts 实现 processNextStart 方法，
2 = {EdgeVertexStep@1410} "EdgeVertexStep(IN)"   和 VertexStep 一样，只是实现了自己的 flatMap 方法。
3 = {PropertiesStep@1411} "PropertiesStep([name],value)" 同上。
```

看到这里我们稍微清楚了一点，还是有很多问题， FlatMapStep 的 processNextStart 逻辑是在干啥我们并没有理清楚。

AbstractStep 有一个 `protected ExpandableStepIterator<S> starts = new ExpandableStepIterator<>(this);` 它的 next 方法如下：

```java
public Traverser.Admin<S> next() {
    if (!this.traverserSet.isEmpty())
        return this.traverserSet.remove();
    /////////////
    if (this.hostStep.getPreviousStep().hasNext())
        return this.hostStep.getPreviousStep().next();
    /////////////
    return this.traverserSet.remove();
}
```

我们可以看出，其实并没有什么逻辑，就是 `return this.hostStep.getPreviousStep().next()`，而前面会加一个 traverserSet 判断，而且 traverserSet 是空的，需要显式调用
add 方法才能添加，所以 ExpandableStepIterator 实际上就是对 Step 的一个拓展代理，让 Step 拥有更多功能。

然后我们回到 FlatMapStep 的 processNextStart 方法，我们可以看出，实际上逻辑就是 flatMap( getPreviousStep().next() ) ，也就是先调用父Step的next方法，然后执行一个 flatMap。
具体的flatMap 方法逻辑由子类实现。

### debug简单总结

通过这次debug我们大概知道了整个程序运行的逻辑。

1. g 是一个 GraphTraversalSource 类型对象，g 作为一个 遍历源每次初始会 new DefaultGraphTraversal<>(GraphTraversalSource)；

2. 首先我们的代码，g.V().outE().inV().values() 每次调用都会添加一个 Step。
例如 g.V() 添加 GraphStep，outE() inV() values() 分别 VertexStep EdgeVertexStep PropertiesStep，他们都继承自 FlatMapStep。step之间有依赖关系。

3. 当调用 Traversal 的 next 等action 方法，会触发查询。实际上会调用 finalEndStep 的next 方法。实际上会调用 AbstractStep 的 processNextStart。
不同的 processNextStart 会有不同的实现，例如 GraphStep 就是查库，FlatMapStep 就是调用上一个 Step 的 next 然后执行 flatMap 方法。类似 spark 的实现方式。


