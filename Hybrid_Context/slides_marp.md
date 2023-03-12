---
marp: true
---

# Hybrid Context-Sensitivity for Points-To Analysis

> 作者：George Kastrinis, Yannis Smaragdakis
> 单位：Department of Informatics University of Athens
> 会议：PLDI'13
> 链接：https://dl.acm.org/doi/pdf/10.1145/2499370.2462191

---

## 1 Intro
Points-to analysis is a `static program analysis` that consists of computing `all objects` (typically identified by allocation site) that `a program variable` may point to.
Points-to analysis is the most standardized and well-understood of `inter-procedural analyses`.
The emphasis of points-to analysis algorithms is on combining fairly `precise modeling` of pointer behavior with `scalability`.

Challenge: how to aproximate?
Allow `satisfactory precision` at a `reasonable cost`.

---

此外，精度提升带来的复杂度提升通常小于理论值，因为精度提升带来的指向集元素更少了。

上下文敏感是指针分析中速度和精度的主要trade-off。
含义：通过上下文信息区分局部变量和对象。
例子参考 [Problem of Context-Insensitive Pointer Analysis](../../2%20课程笔记/静态分析/11%20Pointer%20Analysis%20-%20Context%20Sensitivity%20I.md#Problem%20of%20Context-Insensitive%20Pointer%20Analysis)

---

两种常见的 Context-sensitivity
- call-site-sensitivity (kCFA) [Example](../../2%20课程笔记/静态分析/11%20Pointer%20Analysis%20-%20Context%20Sensitivity%20I.md#Context%20Sensitivity)
- object-sensitivity [Example](../../2%20课程笔记/静态分析/12%20Pointer%20Analysis%20-%20Context%20Sensitivity%20II.md#k-Call-Site%20Sensitivity/k-CFA) 作者觉得叫allocation-site sensitivity更好（x
因为思想上差的比较多，所以原则上这两个没办法比较孰优孰劣。

---

那么能不能有效地结合起来呢？
- 直接结合，比如同时使用1-call-site和1-allocation-site作为上下文，比1-object上下文慢3.9倍。
- 但是，可以根据语言特点在combined context 和 object-only context中切换。比如，static method call和dynamic method call用不同的上下文；或者为程序特征调整上下文定义。
- 但是，static method call中的dynamic method call或者object allocation应该怎么处理呢？
- 结果，他们的混合上下文和naive的混合比精度差不多；速度快很多。而且这个效果可以推广。
- 原因：虽然增加了指针分析的复杂度，但是由于指向关系更准了，所以也减少了一些计算。
- 结论：建立了新的sweet spots: 1-object-sensitive, 2-object-sensitive with a 1context-sensitive heap, and type-sensitive analyses.

---

### Contribution
- 建模了上下文敏感的指针分析参数空间。允许了call-site和object-sensitivity同时存在或者在二者间切换。
- 引入了hybrid call-site- and object-sensitivity的概念。可以自由混合两种不同的上下文，并且根据不同的程序特征来调整混合。目标：保留精确性，成本降低。
- 在DOOP框架上实现了算法。
- 他们的方法跟2-object-sensitive analysis with a context-sensitive heap相比快了1.53倍而且结果更精确了。比1-object-sensitive快了1.12倍，且精度显著增加。 

---

## 2 Modeling of Points-To Analysis
简介指针分析和上下文选择的模型。

---

### 2.1 Background: Parameterizable Model
将指针分析建模成带参数的 [Datalog](../../2%20课程笔记/静态分析/14%20Datalog-Based%20Program%20Analysis.md) 程序。Datalog 的规则单调，不能递归、取反同时存在。

这种方法比较好的模拟了Java和其他高级语言，除了C/C++，因为这些语言会通过&创建指针。建模忽略static field，因为上下文无关。

指针分析 在以下方面 是对实际执行的抽象：
- 实际执行的过多规则。比如，reflection, native methods, static fields, string constants, implicit initialization, threads。本文关注 最关键的9条规则。
- 实际执行对于有效执行的考虑。比如存储模型。The most important is that of defining indexes for the key relations of the evaluation.（有点没看懂

---

![image.png](https://s2.loli.net/2023/03/06/Am3vObdKcXYLVWU.png)

---

上图展示了分析域，输入关系，中间/输出关系，以及三个负责产生上下文的函数。
- 输入关系就是指针分析中的IR
	- LOOKUP匹配方法签名和实际方法定义
	- HEAPTYPE匹配对象到第一个参数的类型
- 5个中间/输出关系，每个方法或者局部变量都有上下文，堆对象有堆上下文。
	- VARPOINTSTO编码指向关系
	- CALLGRAPH编码调用图
	- 其他为了简洁而引入，比如INTERPROCASSIGN统一了static and virtual method calls调用的大部分处理。

---

![image.png](https://s2.loli.net/2023/03/06/BEQAKeM6IrR1Ohw.png)

---

上图展示了指针分析的规则：
- 除了RECORD、MERGE和MERGESTATIC以外的规则，对于不同的上下文敏感分析都适用。即有关上下文的选择都被包含在这三个函数中。其中MERGESTATIC最关键。
	- RECORD创建新的heap上下文，分析ALLOC时才被调用。（上图第三条规则，含义在reachable方法中alloc语句推出指向关系
	- MERGE和MERGESTATIC创建calling上下文(用于局部变量)。含义：在callsite获取信息，创建新的上下文。
- MERGE含义：
	- callerCtx下的可达方法inMeth
	- 有一个操作局部变量base的VCALL
	- 而且到目前为止的分析已经确定base可以指向堆对象heap
	- 确定对象的类型，然后Dispatch实际调用的方法toMeth
	- 传this

---

### 2.2 Instantiating the Model: Standard Analyses
用Datalog讨论heap context和context的变形，就是如上所述在RECORD、MERGE和MERGESTATIC上做文章。
参见 [Context-Sensitive Heap: Example](../../2%20课程笔记/静态分析/11%20Pointer%20Analysis%20-%20Context%20Sensitivity%20I.md#Context-Sensitive%20Heap:%20Example)  

- context-insensitive
```
RECORD (heap,ctx) = * 
MERGE (heap, hctx, invo, ctx) = * 
MERGESTATIC (invo, ctx) = *
```
- 1-call-site-sensitive 即context == callsite
```
RECORD (heap,ctx) = * 
MERGE (heap, hctx, invo, ctx) = invo 
MERGESTATIC (invo, ctx) = invo
```

---

- 1-call-site-sensitive with a context-sensitive heap 和1-callsite类似
	heap context = context = call site
```
RECORD (heap,ctx) = ctx 
MERGE (heap, hctx, invo, ctx) = invo 
MERGESTATIC (invo, ctx) = invo
```

参见 [4 Context Sensitivity Variants](../../2%20课程笔记/静态分析/12%20Pointer%20Analysis%20-%20Context%20Sensitivity%20II.md#4%20Context%20Sensitivity%20Variants)
- 1-object-sensitive (1obj)
```
RECORD (heap,ctx) = *
MERGE (heap, hctx, invo, ctx) = heap
MERGESTATIC (invo, ctx) = ctx
```

---

- 2-object-sensitive with a 1-context-sensitive heap 和1obj类似
```
RECORD (heap,ctx) = first(ctx) // ctx有俩 heapctx就一
MERGE (heap, hctx, invo, ctx) = pair(heap, hctx)
MERGESTATIC (invo, ctx) = ctx
```
对象的heap上下文是allocating method上下文的第一个元素
vitual method的上下文是receiver object和他的heap context(parent recv obj)

- 2-type-sensitive with a 1-context-sensitive heap
```
RECORD (heap,ctx) = first(ctx) // ctx有俩 heapctx就一
MERGE (heap, hctx, invo, ctx) = pair(C_A(heap), hctx)
MERGESTATIC (invo, ctx) = ctx
```
- 其他的组合没有实际意义（效果差）

---

## 3 Hybrid Context-Sensitive Analyses
利用了过去对于什么样的上下文更有效的见解：
- call-site-sensitive heap效果不如object-sensitive heap。与压倒性的成本相比，在call-site-sensitive中加入堆上下文对精度的提升很小。
- 对于面向对象语言来说，object-sensitive更好。

---

### 3.1 Uniform Hybrid Analyses
直接保留两种context，确实精确了，但是增加了成本。

- Uniform 1-object-sensitive hybrid (U-1obj)
用call-site sensitivity增强1object-sensitive，上下文由(C=H\*I)构成
```
RECORD (heap,ctx) = *
MERGE (heap, hctx, invo, ctx) = pair(heap, invo)
MERGESTATIC (invo, ctx) = pair(first(ctx), invo)
```
vitual method的上下文是receiver object和callsite
static method的上下文是caller的context和callsite
由于这种方法的上下文是1-obj的超集，所以肯定更精确。

---

- Uniform 2-object-sensitive with 1-context-sensitive heap hybrid (U-2obj+H).
类似的.
```
RECORD (heap,ctx) = first(ctx) 
MERGE (heap, hctx, invo, ctx) = triple(heap, hctx, invo)
MERGESTATIC (invo, ctx) = triple(first(ctx), second(ctx), invo)
```
值得注意的，这种混合将object放在更加重要的位置上，这种方式产生的堆上下文和(2obj+H)一样。
如果让HC=I，即使用call-site的上下文，不过如前所述，效果更差。（我理解heap明显和obj更加相关。

- Uniform 2-type-sensitive with 1-context-sensitive heap hybrid (U2type+H).
类似的.
```
RECORD (heap,ctx) = first(ctx)
MERGE (heap, hctx, invo, ctx) = triple(C_A(heap), hctx, invo)
MERGESTATIC (invo, ctx) = triple(first(ctx), second(ctx), invo)
```

---

### 3.2 Selective Hybrid Analyses
另一种混合call-site和object-sensitive的方式是在同一个分析中使用不同的上下文。
在选择性上下文中，上下文的集合C和HC会形成笛卡尔积。根据在形成新上下文的不同分析点上的可用信息，将创建不同种类的上下文：
在静态方法调用时，对象敏感分析没有可用的堆对象来创建一个新的上下文，因此它最多只能传播调用者的上下文。然而，callsite可以被用作上下文，区分不同的静态调用。
因为在object-sensitive的分析中，没有太多的信息可以用于创建static method的上下文，所以用call-site当上下文是有好处的。

> 主要是需要想到(发现)static method这种影响指针分析精度的方法，然后给一个解决。
> 就这个方案本身，感觉其实比较trivial的。

---

对于static call的上下文的上下文选择有两种不同的方法：


- Selective 1-object-sensitive hybrid A (S_A-1obj).
keep only a single context element in both virtual and static invocations.
总的上下文域：$C=H\cup I$
```
RECORD (heap,ctx) = *
MERGE (heap, hctx, invo, ctx) = heap
MERGESTATIC (invo, ctx) = invo
```
注意，这个分析并不是1obj的超集，这种方法会表明把static method的上下文从object换成call-site的变化。

---

- Selective 1-object-sensitive hybrid B (SB-1obj).
add extra information to the context of static calls.
vitual method的上下文是object
static method的上下文是caller的上下文+callsite
总的上下文域：$C = H\times (I \cup \{*\})$
```
RECORD (heap,ctx) = *
MERGE (heap, hctx, invo, ctx) = pair(heap,*)
MERGESTATIC (invo, ctx) = pair(first(ctx),invo)
```
这个分析是1obj的超集。

---

- Selective 2-object-sensitive with 1-context-sensitive heap hybrid (S-2obj+H).
用allocation site(object)当heap context
vitual method的上下文不变
总的上下文域：$C = H \times (H \cup I) \times (H \cup I \cup \{*\})$
```
RECORD (heap,ctx) = first(ctx)
MERGE (heap, hctx, invo, ctx) = triple(heap, hctx, *)
MERGESTATIC (invo, ctx) = triple(first(ctx), invo, second(ctx))
```
这种设计对于static method中的static method上下文更倾向于call-site sensitive，有助于产生高质量的堆上下文。

---

- Selective 2-type-sensitive with 1-context-sensitive heap hybrid (S2type+H).
一模一样 略

- 其他策略
比如，heap context也用混合上下文，效果差。
比如，自由选择call-site和object-sensitive的上下文，将有比较好效果的object-sensitive放在了不重要的位置上，也不是一个好的trade-off

---

## 5 Related Work
- 一些不够普遍的混合上下文敏感分析: Java web应用污点分析TAJ
- 流不敏感：但是在SSA的IR上做流不敏感也可以实现流敏感的优点，DOOP也是Jimple，默认无


## 6 Conclusions and Future Work
改进主要来自于上下文在纯object-sensitive上的变化。
1. 提供了更深的上下文敏感分析。
2. 不同call的上下文有不同的形式，不同上下文中的同一个call的上下文有不同的形式。

---

# Thank You