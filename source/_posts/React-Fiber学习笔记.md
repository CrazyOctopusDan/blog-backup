---
title: React-Fiber学习笔记
date: 2019-07-01 16:08:44
tags: React
---

# 介绍
React Fiber 是 React 核心算法的持续重新实现。这是React团队两年多研究的成果。

React Fiber 的目标是提高其适合动画、布局和手势等区域的能力。其主功能是增量渲染:能够将渲染工作拆分为块并将其分散到多个帧上。

其他关键功能包括:
1. 当新更新进来时，可以暂停、中止或重用工作;
2. 能够为不同类型的更新分配优先级;
3. 和新的并发基元。

# 关于这篇文档
`Fiber`引入了几个新颖的概念，单靠查看代码是很难解决的。
这也是一项正在进行中的工作。`Fiber`是一个正在进行的项目，在完成之前可能会进行重大的重构。这篇文档是在这里记录它的设计。

# 先决条件
我强烈建议您在继续之前熟悉以下资源:

*  [React组件、元素和实例](https://reactjs.org/blog/2015/12/18/react-components-elements-and-instances.html) - "组件"通常是一个重载的术语。牢牢把握这些条款至关重要。
*  [协调](https://zh-hans.reactjs.org/docs/reconciliation.html) - React 对帐算法的高级描述。
*  [React基本理论概念](https://github.com/reactjs/react-basic) - 无实现负担的响应概念模型的描述。其中一些在一读时可能没有意义。没关系，随着时间的推移，它更有意义。
*  [React设计原则](https://zh-hans.reactjs.org/docs/design-principles.html) - 特别注意[协调](https://zh-hans.reactjs.org/docs/design-principles.html#scheduling)部分。它做了很好的解释React Fiber的原因。

# 回顾

-------
如果您尚未查看先决条件部分。

在深入探讨新事物之前，让我们先回顾一下概念。

## 什么是协调（reconciliation）?
### 协调（reconciliation）
React 算法用于将一个树与另一个树进行差异，以确定需要更改哪些部分。
### 更新（update）
用于渲染 React 应用的数据的更改。通常是"setState"的结果。最终导致重新渲染。
React 的 API 的核心思想是将更新视为导致整个应用重新渲染。这允许开发人员以声明性方式推理，而不是担心如何有效地将应用从任何特定状态转换到另一种状态(A 到 B，B 到 C，C 到 A，等等)。

实际上，在每个更改上重新渲染整个应用仅适用于最琐碎的应用;在真实应用中，性能成本高得令人望而却步。React 具有优化功能，可创建整个应用重新渲染的外观，同时保持出色的性能。这些优化的大部分都是称为协调的过程的一部分。

协调`Scheduling`是通常理解为"虚拟 DOM"背后的算法，高级描述如下所示:当您渲染 React 应用程序时，将生成描述该应用程序的节点树并将其保存在内存中。然后，此树将刷新到渲染环境 —— 例如，在浏览器应用程序的情况下，它将转换为一组 DOM 操作。更新应用时(通常通过 setState)将生成一个新树。新树与上一个树有差异，以计算更新渲染的应用所需的操作。

尽管`Fiber`是对协调器的一个基础重写，但 React 文档中描述的高级算法将大致相同。要点如下:
* 假定不同的组件类型生成完全不同的树。React 不会尝试将它们分散，而是完全替换旧树。
* 使用key执行列表差异对比的时候，key应具备"稳定、可预测和唯一"的特性。

## 协调与渲染（Reconciliation versus rendering）
DOM 只是 React 可以渲染到的渲染环境之一，其他主要目标是通过响应原生进行本机的本机 iOS 和 Android 视图。(这就是为什么有点用错"虚拟 DOM"。）

它可以支持这么多目标的原因是，React 的设计是使对帐和渲染是单独的阶段。协调者执行计算树的哪些部分已更改的工作;然后，渲染程序使用该信息实际更新渲染的应用。

这种分离意味着 React DOM 和 React Native 可以在共享由 React 核心提供的同一协调器时使用自己的渲染器。

`Fiber` 重新实现协调器。它主要与渲染无关，尽管渲染器需要更改以支持(和利用)新的体系结构。

## 协调（Scheduling）
### 协调
确定何时应执行工作的过程。
### 工作（Work）
必须执行的任何计算。工作通常是更新的结果(例如 setState)。

> 在其当前实现中，React 递归遍历整棵Dom树，并在单个变化期间内调用整个更新树的渲染函数。但是，将来它可能会开始延迟某些更新以避免删除帧。
> 这是 React 设计中的一个常见主题。一些流行的库实现了"Push"方法，其中在新数据可用时就执行计算。但是，React 要坚持"Pull"方法，即计算可以推迟到必要的时刻再进行。
> React 不是通用数据处理库。它是用于构建用户界面的库。我们认为，它在应用中处于独特的位置，可以知道哪些计算现在相关，哪些不相关。
> 如果某些内容位于屏幕外，我们可以延迟与它相关的任何逻辑。如果数据到达速度比帧速率快，我们可以合并和批量更新。我们可以将来自用户交互(如按钮单击引起的动画)的工作优先于不太重要的后台工作(例如渲染刚从网络加载的新内容)，以避免删除帧。

要点如下:

* 在 UI 中，不必立即应用每个更新;事实上，这样做可能会浪费，导致帧下降和降低用户体验。
* 不同类型的更新具有不同的优先级 - 动画更新需要比来自数据存储的更新更快地完成。
* 基于推送的方法要求应用(您，程序员)决定如何安排工作。基于拉取的方法使框架(React)变得智能，并为您做出这些决策。

React 目前没有以明显地表示要使用该计划;一次更新会导致立即重新渲染整个子树。利用协调是 `Fiber` 背后的驱动理念就是重构React的核心算法。

-------
现在，我们已准备好深入了解`Fiber`的实现。接下来提到的内容比我们到目前为止讨论的内容更具有技术性。

# 什么是Fiber？

接下来即将讨论React Fiber架构的核心，`Fiber`是一个比应用程序开发人员通常认为的低级抽象的概念，如果你发现自己在试图理解它时感到沮丧，不要感到气馁。不断尝试，你做的努力最终都会有意义。

-------
我们已经确定 `Fiber` 的主要目标是使 React 能够利用协调。具体来说，我们需要能够

* 暂停工作，然后过一会再回来。
* 为不同类型的工作分配优先级。
* 重用以前完成的工作。
* 如果不再需要操作了，则中止工作。

为了做到这一点，我们首先需要一种方法，将工作分解为单元。从某种意义上说，这就是**纤维（`Fiber`）**。一个纤维表示一个工作单元。

更进一步，让我们回到 React 组件作为数据函数的概念，通常表示为： v = f(d)

因此，呈现 React 应用类似于调用其正文包含对其他函数的调用的函数，等等。这种类比在思考`Fiber`时很有用。

计算机通常跟踪程序执行的方式是使用调用堆栈。执行函数时，将新堆栈帧添加到堆栈中。该堆栈帧表示该函数执行的工作。

在处理 UI 的时候，可能出现问题是，如果一次执行的工作太多，就会导致动画丢弃帧并看起来不连贯。此外，如果某些操作被较新的更新取代，那么这些较旧的操作可能就没有必要。这就是 UI 组件和函数之间的比较大不相同的地方，因为组件比一般函数更具体。

较新的浏览器(和React Native)实现 API，来帮助解决这一确切问题:
请求 IdleCallback 计划在空闲期间调用低优先级函数，并请求动画Frame 计划在下一个调用高优先级函数动画帧。

问题是，为了使用这些 API，工程师们需要一种将渲染工作分解为增量单元的方法。如果仅依赖调用堆栈，它将继续工作，直到堆栈为空。

思考痛点：
* 如果我们能够自定义调用堆栈的行为以优化渲染 UI，这难道不是很棒吗?
* 如果我们能随时中断调用堆栈并手动操作堆栈帧，这难道不是很棒吗?

这就是React Fiber的目的，`Fiber`是堆栈的重新实现，专门用于React组件。您可以将单个`Fiber`视为**虚拟堆栈框架**。

重新实现堆栈的优点是可以将堆栈框架保留在内存中，并在需要时执行它们。这对实现我们实现计划目标至关重要。

除了**协调**之外，手动处理堆栈框架可以释放并发和错误边界等功能的潜力。本文将在之后的部分中介绍这些主题。

在下一节中，文章将更多地介绍`Fiber`的结构。

## Fiber的结构
*注意:随着React对实现细节的更具体的了解，某些内容可能更改的可能性增加，文章内部的内容不能完全代表最新的React Fiber技术*

具体而言，`Fiber`是一个 JavaScript 对象，其中包含有关组件、其输入及其输出的信息。

`Fiber`对应于堆栈框架，但它也对应于组件的实例。

下面是属于`Fiber`的一些重要字段。(此列表并非详尽列出所有字段奥）

### `type` and `key`

`Fiber`的类型和键与"React"元素的用途相同。(事实上，当从元素创建`Fiber`时，这两个字段将直接复制到该字段上。

`Fiber`的类型描述它对应的组件。对于复合组件，类型是函数或类组件本身。对于主机组件(div、span 等)，类型是字符串。

从概念上讲，类型是函数(如 v = f(d))，其执行被堆栈框架跟踪。

与类型一起，键`key`在调节期间使用，以确定是否可以重复使用`Fiber`。

### `child` and `sibling`

这些字段指向其他Fiber,描述了Fiber的递归树结构。

`子Fiber`对应于组件的渲染方法返回的值。因此,在下面的示例中

```Javascript
    function Parent() {
      return <Child />
    }
```

`Parent`的子Fiber(子元素)对应于`Child`。

同级字段用于呈现返回多个子级的情况(`Fiber`中的新功能!):

```Js
    function Parent() {
      return [<Child1 />, <Child2 />]
    }
```

`子Fiber`形成一个单独链接的列表,其头是第一个子元素。因此,在此示例中,`Parent`的子项为`Child1`,`Child1`的同级为`Child2`。

回到我们的函数类比,您可以将`子Fiber`视为 [tail-called function](https://en.wikipedia.org/wiki/Tail_call)

### `return`

`return Fiber`是程序在处理当前`Fiber`后应返回的`Fiber`。它在概念上与堆栈框架的返回地址相同。也可以将其视为`parent Fiber`。

如果` Fiber`有多个`child Fibers`,则每个`child Fiber`的`return Fiber`是`parent Fiber`。因此,在上一节中的示例中,`child 1` 和`child 2`的`return Fiber`是`parent Fiber`。

### `pendingProps` 和`memoizedProps`
从概念上讲,`props`是函数的参数。`Fiber`的`pendingProps`在其执行开始时设置,而`memoizedProps`在末尾设置。

当传入的`pendingProps`等于`memoizedProps`时,它表明`Fiber`以前的输出可以重复使用,从而防止不必要的工作。

### `pendingProps`的优先级
指示由`Fiber`表示的 `props` 的优先级的数字。"React优先级（ReactPriorityLevel）"模块列出了不同的优先级及其表示的内容。

除 NoWork(0)外,数字越大表示优先级较低。例如,可以使用以下函数检查光纤的优先级是否至少与给定级别相同:


```JavaScript
    function matchesPriority(fiber, priority) {
      return fiber.pendingWorkPriority !== 0 &&
             fiber.pendingWorkPriority <= priority
    }
```

此函数仅用于说明;它实际上不是 React Fiber 代码库的一部分。

协调程序（scheduler）使用优先级字段搜索要执行的下一个工作单元。

此算法将在以后的章节中讨论。

### alternate

#### *flush*
刷新（flush）Fiber 是将他的输出渲染在屏幕上。

#### *work-in-progress*
尚未完成的Fiber;从概念上讲,是尚未从堆栈框架返回的 `Fiber`。

在任何时候,组件实例最多具有两个与其对应的 `Fiber`: 当前的 Fiber `the current Fiber`、刷新的Fiber `flushed fiber` 和 尚未完成的Fiber `work-in-progress Fiber`。

`the current Fiber`的备用是`work-in-progress Fiber`,而`work-in-progress Fiber`的备用是`the current Fiber`。

使用称为克隆Fiber `cloneFiber` 的功能 懒创建（ lazily using ） `Fiber`的备用`Fiber`。`cloneFiber` 将尝试重用 `Fiber` 的备用对象`work-in-progress Fiber`(如果存在)而不是始终创建新对象,从而最大限度地减少分配。

工程师们应该将 备用字段 `alternate` 作为实现 `Fiber`的细节,但它在代码库中经常弹出,因此在此处讨论它非常有价值。

### output

#### *host component*

React 应用程序的叶节点。它们特定地存在于渲染环境(例如,在浏览器应用中,它们是"div","span"等)。在 JSX 中,它们使用 **小写的名称标签** 表示。

从概念上讲,`Fiber`的输出是函数的返回值。

每个`Fiber`最终都有输出,但输出仅由**host component**在叶节点上创建，这个输出然后再向树上转移。

最终提供给渲染器的输出的内容将刷新渲染到展示的环境中。渲染器有责任定义这个最终输出的内容是应该 创建 还是 更新。

# 未来部分
这就是现在所有的一切,但本文档还远远不够完整。后续部分将介绍更新整个生命周期中使用的算法。要涵盖的主题包括:

* 协调程序（scheduler）如何找到要执行的下一个工作单元。
* 如何通过 Fiber 树跟踪和传播优先级。
* 计划程序如何知道何时暂停和恢复工作。
* 如何刷新工作并标记为已完成。
* 副作用(如生命周期方法)的工作原理。
* 什么是协同例程,以及如何使用它来实现上下文和布局等功能。

# 参考文献
[Fiber](https://github.com/acdlite/react-fiber-architecture)
[Virtual DOM 及内核](https://zh-hans.reactjs.org/docs/faq-internals.html#what-is-react-fiber)
