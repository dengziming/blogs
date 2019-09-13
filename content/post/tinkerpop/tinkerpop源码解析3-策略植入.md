---
date: 2018-10-30
title: "tinkerpop源码解析3-策略植入"
author: "邓子明"
tags:
    - tinkerpop
    - 知识图谱
categories:
    - 源码
comment: true
---

前面我们大概看了tinkerpop 的代码怎么一步步变成 Traversal 和 Step，然后怎么调用和执行。我们忽略了 strategy 的相关操作。

## 官方介绍

在查看源码之前我们尽量对她进行了解熟悉。TraversalStrategy 分析 Traversal，如果 Traversal 符合其标准尺度，则可以相应地进行改变。
strategies 在编译期执行，是 Gremlin traversal machine 编译器的基础，分为五种类型：
1. 可以直接嵌入 traversal 逻辑的 application-level 的功能（decoration）；
2. 能够更高效通过 TinkerPop3 语法表达 traversal （optimization）；
3. 能够更高效在 graph 层面表达traversal（provider optimization）；
4. 在执行 traversal 之前 一些必须的最终适配、清理、分析 (finalization)；
5. 对于当前的程序或者存储系统来说，一些不合法的操作(verification)。

以 IdentityRemovalStrategy 为例：

```java
public final class IdentityRemovalStrategy extends AbstractTraversalStrategy<TraversalStrategy.OptimizationStrategy> implements TraversalStrategy.OptimizationStrategy {

    private static final IdentityRemovalStrategy INSTANCE = new IdentityRemovalStrategy();

    private IdentityRemovalStrategy() {
    }

    @Override
    public void apply(Traversal.Admin<?, ?> traversal) {
        if (traversal.getSteps().size() <= 1)
            return;

        for (IdentityStep<?> identityStep : TraversalHelper.getStepsOfClass(IdentityStep.class, traversal)) {
            if (identityStep.getLabels().isEmpty() || !(identityStep.getPreviousStep() instanceof EmptyStep)) {
                TraversalHelper.copyLabels(identityStep, identityStep.getPreviousStep(), false);
                traversal.removeStep(identityStep);
            }
        }
    }

    public static IdentityRemovalStrategy instance() {
        return INSTANCE;
    }
}
```
可以看出继承自 AbstractTraversalStrategy， 和 TraversalStrategy.OptimizationStrategy，并有一个 apply(Traversal.Admin<?, ?> traversal)  方法。
这里是直接在 traversal 中找到 identityStep 并移除。


## applyStrategies

strategies 初始化的时候就放入graph 中，我们这里看看如何嵌入到查询语句。

在我们调用 traversal.next() 之前，有个步骤就是 applyStrategies。在执行 traversal 之前大概是这样的：
```java
this = {DefaultGraphTraversal@1359} "[GraphStep(vertex,[1]), VertexStep(OUT,[knows],edge), EdgeVertexStep(IN), PropertiesStep([name],value)]"
 lastTraverser = {EmptyTraverser@1384} 
 finalEndStep = {EmptyStep@1385} 
 stepPosition = {StepPosition@1362} "4.0.0()"
 graph = {TinkerGraph@1386} "tinkergraph[vertices:6 edges:6]"
 steps = {ArrayList@1387}  size = 4
 unmodifiableSteps = {Collections$UnmodifiableRandomAccessList@1388}  size = 4
 parent = {EmptyStep@1385} 
 sideEffects = {DefaultTraversalSideEffects@1389} "sideEffects[size:0]"
 strategies = {DefaultTraversalStrategies@1361} "strategies[ConnectiveStrategy, IncidentToAdjacentStrategy, MatchPredicateStrategy, FilterRankingStrategy, InlineFilterStrategy, AdjacentToIncidentStrategy, RepeatUnrollStrategy, PathRetractionStrategy, CountStrategy, LazyBarrierStrategy, TinkerGraphCountStrategy, TinkerGraphStepStrategy, ProfileStrategy, StandardVerificationStrategy]"
 generator = null
 requirements = null
 locked = false
 bytecode = {Bytecode@1390} "[[], [V(1), outE(knows), inV(), values(name)]]"
 ```

可以看出是 DefaultGraphTraversal 的实体类，lastTraverser finalEndStep parent sideEffects 都是刚初始化，steps unmodifiableSteps 是4个。
然后我们执行完这个方法再来看看：

```java
this = {DefaultGraphTraversal@1359} "[TinkerGraphStep(vertex,[1]), VertexStep(OUT,[knows],vertex), PropertiesStep([name],value)]"
 lastTraverser = {EmptyTraverser@1384} 
 finalEndStep = {PropertiesStep@1432} "PropertiesStep([name],value)"
 stepPosition = {StepPosition@1362} "6.0.0()"
 graph = {TinkerGraph@1386} "tinkergraph[vertices:6 edges:6]"
 steps = {ArrayList@1387}  size = 3
 unmodifiableSteps = {Collections$UnmodifiableRandomAccessList@1388}  size = 3
 parent = {EmptyStep@1385} 
 sideEffects = {DefaultTraversalSideEffects@1389} "sideEffects[size:0]"
 strategies = {DefaultTraversalStrategies@1361} "strategies[ConnectiveStrategy, IncidentToAdjacentStrategy, MatchPredicateStrategy, FilterRankingStrategy, InlineFilterStrategy, AdjacentToIncidentStrategy, RepeatUnrollStrategy, PathRetractionStrategy, CountStrategy, LazyBarrierStrategy, TinkerGraphCountStrategy, TinkerGraphStepStrategy, ProfileStrategy, StandardVerificationStrategy]"
 generator = null
 requirements = {Collections$UnmodifiableSet@1431}  size = 1
 locked = true
 bytecode = {Bytecode@1390} "[[], [V(1), outE(knows), inV(), values(name)]]"
```

finalEndStep 变了，stepPosition 变了，steps unmodifiableSteps 都少了一步，requirements 多了一个。然后我们还是具体看看方法执行步骤：

```java
public void applyStrategies() throws IllegalStateException {
    if (this.locked) throw Traversal.Exceptions.traversalIsLocked(); // 判断是否执行过。
    TraversalHelper.reIdSteps(this.stepPosition, this);
    this.strategies.applyStrategies(this); // 这一步循环调用 strategy 的 apply 方法。
    {
        for (final TraversalStrategy<?> traversalStrategy : this.traversalStrategies) {
            traversalStrategy.apply(traversal);
        }
    }
    boolean hasGraph = null != this.graph;
    
    // 判断 TraversalParent 的所有 globalChild 也要进行 applyStrategies
    for (int i = 0, j = this.steps.size(); i < j; i++) { // "foreach" can lead to ConcurrentModificationExceptions
        final Step step = this.steps.get(i);
        if (step instanceof TraversalParent) {
            for (final Traversal.Admin<?, ?> globalChild : ((TraversalParent) step).getGlobalChildren()) {
                globalChild.setStrategies(this.strategies);
                globalChild.setSideEffects(this.sideEffects);
                if (hasGraph) globalChild.setGraph(this.graph);
                globalChild.applyStrategies();
            }
            for (final Traversal.Admin<?, ?> localChild : ((TraversalParent) step).getLocalChildren()) {
                localChild.setStrategies(this.strategies);
                localChild.setSideEffects(this.sideEffects);
                if (hasGraph) localChild.setGraph(this.graph);
                localChild.applyStrategies();
            }
        }
    }
    // 得到 finalEndStep 
    this.finalEndStep = this.getEndStep();
    // finalize requirements
    if (this.getParent() instanceof EmptyStep) {
        this.requirements = null;
        this.getTraverserRequirements();
    }
    this.locked = true;
}
```

整段代码并不复杂，复杂的是每个 traversalStrategy 具体都做了什么。

我们知道 tinkerpop 是一个独立的项目，里面是不带任何和 janus 相关的内容，如何会有 JanusGraphStep 呢，
其实在很多地方（例如 fill:175, Traversal (org.apache.tinkerpop.gremlin.process.traversal)）会调用 this.asAdmin().applyStrategies();
会循环调用 traversalStrategy.apply(traversal)。在 apply 方法中得到了所有的 GraphStep。然后 new JanusGraphStep ，并且替换掉原有的 step。

