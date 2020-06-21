---
title: React源码学习
date: 2020-06-11 11:40:26
tags: React
---

# React-dom 源码阅读
## react-dom-render
首先 => 创建
ReactDOM.render
### 步骤
1. 创建ReactRoot
2. 创建FiberRoot和RootFiber
3. 创建更新

通过DOM `Renderer.createContainer` 创建了一个 `FiberRoot`；

接下来挂载到`container._reactRootContainer`上面；

然后最重要的是，调用 `DOMRenderer.unbatchedUpdates`（批量更新），修改了scheduler里面的一个全局变量；

返回了一个`root.render(children, callback)` => 其实这里的操作最重要是调用了`DOMRenderer.updateContainer(children, root, null, work._onCommit)` => 关键函数`updateContainerAtExpirationTime` 这里有个关键参数是 expirationTime （有效期限？失效时间？）=> `scheduleRootUpdate` 中使用 `createUpdate`标记更新的位置，使用`enqueueUpdate`批量更新，使用`scheduleWork`调度工作（因为react16有任务优先级）

### 关键性函数
首先在ReactDOM.js中找到 ReactDOM 对象 中的 render
在 render 的方法中仔细分析 `legacyRenderSubtreeIntoContainer(null, element, container, false, callback)`

第一个参数是 parentComponent ，接着往下是找到root，
然而在新建的时候一定不会有reactRootContainer，所以创建的时候就只需要关注`!root`

第四个参数是false，跟 hydrate 相关，这个在服务端渲染中有用，告诉react是否可以复用别的节点。

### FiberRoot
* 整个应用的起点
* 包含应用挂载的目标节点
* 记录整个应用更新过程的各种信息

[数据结构](https://react.jokcy.me/book/api/react-structure.html)

### Fiber
* 每一个ReactElement都会对应一个Fiber对象
* 记录节点的各种状态
* 串联整个应用形成树结构

[数据结构](https://react.jokcy.me/book/api/react-structure.html)

串联整个应用形成树结构：
```
{
    ...
    return,   // 返回父节点
    child,    // 第一个子节点
    sibling   // 第一个子节点的兄弟节点
    ...
}
```
只存一个child（第一个子节点），其他的子节点都是第一个child的sibling

等等等

### Update

用于记录组件状态的改变
存放于UpdateQueue中
多个Update可以同时存在

[数据结构](https://react.jokcy.me/book/api/react-structure.html)


