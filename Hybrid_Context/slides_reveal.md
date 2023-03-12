---
title: Hybrid Context-Sensitivity for Points-To Analysis
separator: <!--s-->
verticalSeparator: <!--v-->
theme: simple
highlightTheme: github
css: custom.css
revealOptions:
    transition: 'slide'
    transitionSpeed: fast
    center: false
    slideNumber: "c/t"
    width: 1600
    height: 900
    margin: 0.04
---

<div class="center">
<div style="width: 100%; margin-top: 200px;">

# Hybrid Context-Sensitivity for Points-To Analysis

</div>
</div>

<div style="width: 100%; margin-top: 150px;">

> 作者：George Kastrinis, Yannis Smaragdakis
> 
> 单位：Department of Informatics University of Athens
>
> 会议：PLDI'13
> 
> 链接：https://dl.acm.org/doi/pdf/10.1145/2499370.2462191

</div>

<!--s-->

<div class="middle center">
<div style="width: 100%">

# 1 Intro

</div>
</div>


<!--v-->

## 什么是指针分析

</br>

- Points-to analysis is a **static program analysis** that consists of computing **all objects** (typically identified by allocation site) that **a program variable** may point to.

- Points-to analysis is the most standardized and well-understood of **inter-procedural analyses**.

- The emphasis of points-to analysis algorithms is on combining fairly **precise modeling** of pointer behavior with **scalability**.

</br>

### 挑战: 如何近似?
Allow **satisfactory precision** at a **reasonable cost**.

<!--v-->

## 上下文敏感

<div class="mul-cols">
<div class="col">

上下文敏感是指针分析中速度和精度的最主要的trade-off。

含义：通过上下文信息区分局部变量和对象。

上下文不敏感在一个方法有多个调用的时候，由于不区分调用，所以会出现精度上的丢失。

如右图所示例子中，由于n1和n2都会流到n，就不准了。

#### 两种常见的上下文敏感

- call-site-sensitivity (kCFA)(抽象调用栈)
- object-sensitivity

由于两种上下文吗敏感思想上差的比较多，所以原则上这两个没办法比较孰优孰劣。

</div>

<div class="col">

<a href="https://sm.ms/image/f6yZCdHm8ezlP9a" target="_blank"><img src="https://s2.loli.net/2023/03/12/f6yZCdHm8ezlP9a.png" width="100%"></a>

</div>
</div>

<!--v-->

## 所以？
既然两种方法差的比较多，能不能有效的结合起来取长补短呢？

- 直接结合，比如同时使用1-call-site和1-allocation-site作为上下文，比1-object上下文慢3.9倍。
- 但是，可以根据语言特点在combined context 和 object-only context中切换。比如，static method call和dynamic method call用不同的上下文；或者为程序特征调整上下文定义。
- 但是，static method call中的dynamic method call或者object allocation应该怎么处理呢？
- 结果，他们的混合上下文和naive的混合比精度差不多；速度快很多。而且这个效果可以推广。
- 原因：虽然增加了指针分析的复杂度，但是由于指向关系更准了，所以也减少了一些计算。
- 结论：建立了新的sweet spots: 1-object-sensitive, 2-object-sensitive with a 1context-sensitive heap, and type-sensitive analyses.

<!--v-->

## Contribution
- 建模了上下文敏感的指针分析参数空间。允许了call-site和object-sensitivity同时存在或者在二者间切换。
- 引入了hybrid call-site- and object-sensitivity的概念。可以自由混合两种不同的上下文，并且根据不同的程序特征来调整混合。目标：保留精确性，成本降低。
- 在DOOP框架上实现了算法。
- 他们的方法跟2-object-sensitive analysis with a context-sensitive heap相比快了1.53倍而且结果更精确了。比1-object-sensitive快了1.12倍，且精度显著增加。 


<!--s-->

<div class="middle center">
<div style="width: 100%">

# 2 Modeling of Points-To Analysis
简介指针分析和上下文选择的模型。

</div>
</div>

<!--v-->

## 2.1 DataLog

Datalog: 逻辑式声明编程语言，是Prolog的子集。

最开始是数据库查询语言，现在在程序分析、大数据等领域有应用。

Datalog = Data + Logic
- 没有副作用（变量赋值
- 没有控制流
- 没有函数
- 不是图灵完备的
- 支持递归❗️
- 谓词+规则
- 只需要写指针分析的规则，不需要实现具体算法

```datalog
Reach(from, to) <- 
	Edge(from, to). 
	
Reach(from, to) <- 
	Reach(from, node), 
	Edge(node, to).
```

<!--v-->

## 2.2 参数化模型

<div class="mul-cols">
<div class="col">

<a href="https://sm.ms/image/XSN19VvetAEi8DT" target="_blank"><img src="https://s2.loli.net/2023/03/12/XSN19VvetAEi8DT.png" ></a>

- 输入关系就是指针分析中的IR
  - LOOKUP匹配方法签名和实际方法定义(Dispatch)

</div>
<div class="col">

<a href="https://sm.ms/image/G3tNp6TReQV1AEj" target="_blank"><img src="https://s2.loli.net/2023/03/12/G3tNp6TReQV1AEj.png" ></a>

- 5个中间/输出关系，每个方法或者局部变量都有上下文，堆对象有堆上下文。
  - VARPOINTSTO 编码指向关系
  - CALLGRAPH 编码调用图

</div>
</div>

<!--v-->

## 2.2 参数化模型

<div class="mul-cols">
<div class="col">

<a href="https://sm.ms/image/F1PgE7GfUTDxjh4" target="_blank"><img src="https://s2.loli.net/2023/03/12/F1PgE7GfUTDxjh4.png" ></a>



</div>
<div class="col">

<a href="https://sm.ms/image/lVvQTPZkOSt7WNK" target="_blank"><img src="https://s2.loli.net/2023/03/12/lVvQTPZkOSt7WNK.png" ></a>

</div>
</div>

<!--v-->

## 规则含义

- 除了RECORD、MERGE和MERGESTATIC以外的规则，对于不同的上下文敏感分析都适用。即有关上下文的选择都被包含在这三个函数中。其中MERGESTATIC最关键。
	- RECORD创建新的heap上下文，分析ALLOC时才被调用。（上图第三条规则，含义在reachable方法中alloc语句推出指向关系
	- MERGE和MERGESTATIC创建calling上下文(用于局部变量)。含义：在callsite获取信息，创建新的上下文
    - 为什么变量和heap都需要上下文，参见笔记11
</br>

- MERGE含义：
	- callerCtx下的可达方法inMeth
	- 有一个操作局部变量base的VCALL
	- 而且到目前为止的分析已经确定base可以指向堆对象heap
	- 确定对象的类型，然后Dispatch实际调用的方法toMeth
	- 传this

<!--v-->

## 2.3 建模传统分析

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
- 1-call-site-sensitive with a context-sensitive heap 和1-callsite类似
	
    heap context = context = call site
```
RECORD (heap,ctx) = ctx 
MERGE (heap, hctx, invo, ctx) = invo 
MERGESTATIC (invo, ctx) = invo
```

<!--v-->

## 2.3 建模传统分析

- 1-object-sensitive (1obj)
```
RECORD (heap,ctx) = *
MERGE (heap, hctx, invo, ctx) = heap
MERGESTATIC (invo, ctx) = ctx
```
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

<!--s-->

<div class="middle center">
<div style="width: 100%">

# 3 Hybrid Context-Sensitive Analyses

</br>

利用了过去对于什么样的上下文更有效的见解：

</br>

- call-site-sensitive heap效果不如object-sensitive heap。与压倒性的成本相比，在call-site-sensitive中加入堆上下文对精度的提升很小。

</br>

- 对于面向对象语言来说，object-sensitive更好。

</div>
</div>

<!--v-->


## 3.1 Uniform Hybrid Analyses
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

- Uniform 2-object-sensitive with 1-context-sensitive heap hybrid (U-2obj+H).类似的.
```
RECORD (heap,ctx) = first(ctx) 
MERGE (heap, hctx, invo, ctx) = triple(heap, hctx, invo)
MERGESTATIC (invo, ctx) = triple(first(ctx), second(ctx), invo)
```

值得注意的，这种混合将object放在更加重要的位置上，这种方式产生的堆上下文和(2obj+H)一样。

如果让HC=I，即使用call-site的上下文，不过如前所述，效果更差。（我理解heap明显和obj更加相关。

<!--v-->

## 3.2 Selective Hybrid Analyses

另一种混合call-site和object-sensitive的方式是在同一个分析中使用不同的上下文。

在选择性上下文中，上下文的集合C和HC会形成笛卡尔积。根据在形成新上下文的不同分析点上的可用信息，将创建不同种类的上下文：
- 在static method被调用时，对象敏感分析没有可用的堆对象来创建一个新的上下文，因此它最多只能传播调用者的上下文。然而，callsite可以被用作上下文，区分不同的静态调用。
- 因为在object-sensitive的分析中，没有太多的信息可以用于创建static method的上下文，所以用call-site当上下文是有好处的。

<!--v-->

## 3.3 两种 static method 的上下文选择

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

<!--v-->

## 3.3 两种 static method 的上下文选择

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

<!--v-->

## 3.4 其他选择性上下文策略

- Selective 2-object-sensitive with 1-context-sensitive heap hybrid (S-2obj+H).

用allocation site(object)当heap context

vitual method的上下文不变

总的上下文域：$C = H \times (H \cup I) \times (H \cup I \cup \{*\})$
```
RECORD (heap,ctx) = first(ctx)
MERGE (heap, hctx, invo, ctx) = triple(heap, hctx, *)
MERGESTATIC (invo, ctx) = triple(first(ctx), invo, second(ctx))
```
这种设计对于static method中的static method上下文更倾向于**call-site sensitive**，有助于产生高质量的堆上下文。

</br>

- 其他策略

比如，heap context也用混合上下文，效果差。

比如，自由选择call-site和object-sensitive的上下文，将有比较好效果的object-sensitive放在了不重要的位置上，也不是一个好的trade-off

<!--s-->

<div class="middle center">
<div style="width: 100%">

# 4 Conclusions
改进主要来自于上下文在纯object-sensitive上的变化。
1. 提供了更深的上下文敏感分析。
2. 不同call的上下文有不同的形式，不同上下文中的同一个call的上下文有不同的形式。

</div>
</div>


<!--s-->

<div class="middle center">
<div style="width: 100%">

# Thank You

</div>
</div>