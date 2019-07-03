---
title: React虚拟DOM 深入学习
date: 2019-05-1 20:10:04
tags: React
top: True
---

## 深入理解 React 虚拟DOM
### 为什么我需要（React-virtual-dom）？

产品的功能源自需求，react 作为一个成功的UI库也是如此。

假象我们有这种需求：
> 我需要自己撸一个轮子，能够在数据改变的时候，及时相应在页面上面，怎么做？

一个工作两年左右的工程师就会思考：
1. state改变监听
2. 拥有一个JSX模板
3. state改变被监听+模板变化 = 生成一个DOM展示；
4. state又变了；
5. state改变被监听+模板变化 = 生成一个新的DOM展示；

问题存在吗？

问题很大：
* 第一次生成了一整个DOM；
* 第二次有生成了一整个DOM;
* 第二次的DOM完整替换了第一次生成的DOM;

重绘很耗费性能的好嘛。

粗略地改进：
...4
5. 在替换旧的DOM之前进行对比，不一样的再替换之（实际上就是DocumentFragment）;
6. 展示新的DOM;

提升如何？
其实并不明显

<!-- more -->

React方案改进
...2
3. state改变被监听+模板变化 => 生成虚拟DOM，用以描述真实DOM；
4. 用虚拟DOM生成真实DOM；
5. state又变了；
6. state改变被监听+模板变化 => 生成新的虚拟DOM，比较旧的虚拟DOM和新鲜的虚拟DOM，找到区别，直接修改；

终于：
* 性能提升一截；
* 甚至使得跨端应用成为现实，RN（React Native）得到广泛使用。为什么呢？因为原生应用里不存在DOM这个概念，
但是虚拟DOM这个思想，这个简单的js对象是完全可以正常跑通的，因此，只需要在执行环境中判断出是在Browser还是App中，就可以使React开发多端应用。

### 都9102年了，你凭什么还说我们浏览器慢？
先来看一看浏览器加载HTML文件都需要做什么：
![BrowserPaint](http://ww3.sinaimg.cn/large/006tNc79ly1g4evhx81ttj30ho089dgf.jpg)

虽然每个浏览器有自己的引擎，但是大致工作流程都差不多，如图所示分为5步：
1. 用HTML分析器（解析器），分析HTML元素，构建一棵**DOM 树**；
2. 用CSS分析器（解析器），分析CSS文件和元素的inline样式，生成**页面样式表**；
3. 将前面的**DOM树**和**样式表**结合，构建一棵**Render树**。这个过程被称为Attachment，每一个DOM节点都拥有attach方法，接收样式信息，返回一个render对象（renderer)。通过这些render对象(renderer)构成一棵**Render树**；
4. 浏览器对着**Render树**开始布局，为每一个存在于Render树上的节点确定一个精确的坐标；
5. Render树有了，节点显示的位置坐标也有了，最后一步就是调用每个节点的**paint**方法，让他们现出原形！

那么，当你在撸代码的时候，使用了原生的api，或者是JQuery（老夫不管……）去操作DOM时，浏览器就会按照以上5个步骤重新走一遍，而且更加恐怖的是，假设你在写代码时，“我要一次性更新10个DOM节点”，那在浏览器眼中，它收到第一个更新请求后并不知道后面还有九个，因此立刻执行（1.2.3.4.5.），最终执行了10遍流程。
假如说每一次新的更新都会对前一个DOM节点的坐标值发生影响，那么也就是说，只有最后一次计算节点的坐标才是有用的，前面9次白算！
所以说，虽然随着时代的变迁，计算机硬件不停地升级，Mac电脑都卖到2W了，但是操作DOM的代价依然非常昂贵，频繁地操作更会出现卡顿等奇奇怪怪の现象，严重影响用户体验。
不要反驳说，我一个小小的简单的DOM节点能有多少玩意儿啊？
给你看看
![](http://ww2.sinaimg.cn/large/006tNc79ly1g4ew0zde6ej32740jganl.jpg)

那怕是一个小小的div，都有如此多的属性，那么整个DOM树有多少，想想都害怕。

### 什么是虚拟Dom呢？
虚拟DOM是一个描述真实DOM的简单js对象

回到上面那个问题中，加入一次操作中有10个更新DOM的操作，那么我虚拟DOM不会像普通浏览器那样傻傻的立即操作，而是将10次更新内容的diff保存到本地的一个js对象中，最终将这个js对象一次性attach到DOM树上，通知浏览器，你好去paint了，这样就可以极大程度上节约计算成本，避免无谓的计算，好钢要用在刀刃上。
```Javascript
const VirtualDomTree = Element('div', { id: 'virtual-container' }, [
    Element('p', {}, ['Virtual DOM']),
    Element('div', {}, ['before update']),
    Element('ul', {}, [
        Element('li', { class: 'list-item' }, ['Item 1']),
        Element('li', { class: 'list-item' }, ['Item 2']),
        Element('li', { class: 'list-item' }, ['Item 3']),
    ])
])
```

好处很明显，数据的更新第一步反映在**js对象**上，在内存中对**js对象**的操作速度肯定比浏览器慢悠悠跑来的快得多，等到更新完毕，再交由浏览器去绘制，完美。

那再具体一点，到底是怎么实现的呢？

```Javascript
/**
* @param tagName 节点名
* @param props 节点的属性
* @param children 子节点
**/
function Element(tagName, props, children) {
    if (!(this instanceof Element)) {
        return new Element(tagName, props, children);
    }

    this.tagName = tagName;
    this.props = props || {};
    this.children = children || [];
    this.key = props ? props.key : undefined;

    let count = 0;
    this.children.forEach((child) => {
        if (child instanceof Element) {
            count += child.count;
        }
        count++;
    });
    this.count = count;
}
```

> 除了以上三个参数之外还会保存key和count

OK，到了这一步还没有结束，等到有了js对象之后，还需要将其映射成为真实的DOM：

```Javascript
    Element.prototype.render = function() {
        const el = document.createElement(this.tagName);
        const props = this.props;

        for (const propName in props) {
            setAttr(el, propName, props[propName]);
        }

        this.children.forEach((child) => {
            const childEl = (child instanceof Element) ?
                            child.render() : document.createTextNode(child);
            el.appendChild(childEl);
        });

        return el;
    };
```

根据DOM名调用源生的createElement创建真实DOM，将DOM的属性全都加到这个DOM元素上，如果有子元素继续递归调用创建子元素，并appendChild挂到该DOM元素上。这样就完成了从创建虚拟DOM到将其映射成真实DOM的全部工作。

### 神秘的Diff算法

这个玩意儿算是面试总会问到的。

两棵树如果进行完全的比较，那么时间复杂度是O(n^3 );
但是通过《深入浅出React和Redux》这本书的介绍，Diff算法的时间复杂度只有O(n)！

要实现如此低的时间复杂度，那么就要牺牲一些东西，比如，深度遍历、精确性。
但是，在现实的前端开发中，跨层级的DOM元素的操作不是占据大多数情形的，因此，这么选择的Diff算法是最优的。

![](http://ww4.sinaimg.cn/large/006tNc79ly1g4eyqxkkccj31g00gmwh0.jpg)

diff算法中只会比较同层级的元素，一旦发现某一级之间有所不同，则会弃置其子级，直接用从新的差异的一级以及其下的所有子级替换旧的。我们会有个疑问，这样做那子级中相同的元素不是无法复用了吗，那怎么还能提高比对性能？这无疑是一种缺陷，但也带来了好处就是算法实现简单，也就提高了比对速度，因此最后也是能够提升性能的。

现在，某张小熠又新创建了一棵树，用于和上文中的树比较：
```Javascript
    const newVirtualDomTree = Element('div', { id: 'virtual-container' }, [
        Element('h3', {}, ['Virtual DOM']),                     // REPLACE
        Element('div', {}, ['after update']),                   // TEXT
        Element('ul', { class: 'newList' }, [              // PROPS
            Element('li', { class: 'item' }, ['Item 1']),
            // Element('li', { class: 'item' }, ['Item 2']),    // REORDER remove
            Element('li', { class: 'item' }, ['Item 3']),
        ]),
    ]);
```
![](http://ww3.sinaimg.cn/large/006tNc79ly1g4ez87s2x1j30z409i75b.jpg)
花里胡哨的线，但是也大概表达了Diff了些什么：
* 第一种（蓝色）是最简单的，如图中的p标签变成了H3标签，这个过程被称之为 ***_REPLACE_*** 旧节点包括下面的子节点都将被卸载，如果新节点和旧节点仅仅是类型不同，但下面的所有子节点都一样时，这样做显得效率不高。但为了避免O(n^3 )的时间复杂度，这样做是值得的。这也提醒了React开发者，应该避免无谓的节点类型的变化，例如运行时将div变成p就没什么太大意义。
* 第二种（紫色）也比较简单，节点类型一样，仅仅属性或属性值变了。

```Javascript
    renderA: <ul>
    renderB: <ul class: 'newList'>
    => [addAttribute class "newList"]
```
这个过程被称为 ***_PROPS_***
这个时候不会触发节点的卸载（componentWillUnmounted）和装载（componentWillMount）生命周期，而是执行了 **节点更新** （shouldComponentUpdated => componentDidUpdated 系列方法）


```JavaScript
    function diffProps(oldNode, newNode) {
        const oldProps = oldNode.props;
        const newProps = newNode.props;

        let key;
        const propsPatches = {};
        let isSame = true;

        // find out different props
        for (key in oldProps) {
            if (newProps[key] !== oldProps[key]) {
                isSame = false;
                propsPatches[key] = newProps[key];
            }
        }

        // find out new props
        for (key in newProps) {
            if (!oldProps.hasOwnProperty(key)) {
                isSame = false;
                propsPatches[key] = newProps[key];
            }
        }

        return isSame ? null : propsPatches;
    }
```

* 第三种（绿色）就只是文本变化了，文本其实也是一个Text Node，这个简单，直接修改文字内容即可，被称之为 ***_TEXT_***
* 第四种是 移动、增加、删除子节点，这个过程被称之为 ***_REORDER_***

具体可以看 [虚拟DOM 算法解析](https://www.infoq.cn/article/react-dom-diff/)

#### 列表渲染的元素，你如果不加key，我就嗷嗷叫……

![](http://ww3.sinaimg.cn/large/006tNc79ly1g4ezut9k77j310a02uwfs.jpg)
这个warning，vue和react都会报。他们强烈建议开发者，拜托你在写通过数组循环渲染item的时候，一定要加上key，不然我们在虚拟DOM比较的时候就只能进行两层循环，才知道什么发生改变了，你们开发者如果加上了key，那我们就可以非常快速且清晰地比较出新增和删除了什么东西！

比如⤵️

A B 【F】 C D E

我想要插入一个F元素，那么简单粗暴的方法就出现了：
卸载C，装载F，卸载D，装载C，卸载E，装载D，装载E
![](http://ww1.sinaimg.cn/large/006tNc79ly1g4f02dkxxcj30su09w757.jpg)

如果我们在JSX里为数组或枚举型元素增加上key后，React就能根据key，直接找到具体的位置进行操作，效率比较高。

>  Keys should be "stable, predictable, and unique." 所以不建议在使用key的时候，简单地使用上数组的index属性，那个玩意儿会带来巨大的坑。

![](http://ww1.sinaimg.cn/large/006tNc79ly1g4f08h182lj30s609ejsa.jpg)

> 因此就变成了最小编辑距离问题，可以用Levenshtein Distance算法来实现，时间复杂度是O(M*N)，但通常我们只要一些简单的移动就能满足需要，降低点精确性，将时间复杂度降低到O(max(M, N)即可。

最终Diff出来的结果

```JavaScript
    {
        1: [ {type: REPLACE, node: Element} ],
        4: [ {type: TEXT, content: "after update"} ],
        5: [ {type: PROPS, props: {class: "newList"}}, {type: REORDER, moves: [{index: 2, type: 0}]} ],
        6: [ {type: REORDER, moves: [{index: 2, type: 0}]} ],
        8: [ {type: REORDER, moves: [{index: 2, type: 0}]} ],
        9: [ {type: TEXT, content: "Item 3"} ],
    }
```

### 最终映射到真实DOM中

深度遍历DOM将Diff的内容更新进去：

```Javascript
    function dfsWalk(node, walker, patches) {
        const currentPatches = patches[walker.index];

        const len = node.childNodes ? node.childNodes.length : 0;
        for (let i = 0; i < len; i++) {
            walker.index++;
            dfsWalk(node.childNodes[i], walker, patches);
        }

        if (currentPatches) {
            applyPatches(node, currentPatches);
        }
    }
```

具体的更新代码如下
其实就是根据Diff信息调用源生API操作DOM：


```JavaScript
    function applyPatches(node, currentPatches) {
        currentPatches.forEach((currentPatch) => {
            switch (currentPatch.type) {
                case REPLACE: {
                    const newNode = (typeof currentPatch.node === 'string')
                        ? document.createTextNode(currentPatch.node)
                        : currentPatch.node.render();
                    node.parentNode.replaceChild(newNode, node);
                    break;
                }
                case REORDER:
                    reorderChildren(node, currentPatch.moves);
                    break;
                case PROPS:
                    setProps(node, currentPatch.props);
                    break;
                case TEXT:
                    if (node.textContent) {
                        node.textContent = currentPatch.content;
                    } else {
                        // ie
                        node.nodeValue = currentPatch.content;
                    }
                    break;
                default:
                    throw new Error(`Unknown patch type ${currentPatch.type}`);
            }
        });
    }
```

这个时候呼应开头了，虚拟DOM的目的是将所有操作累加起来，统计计算出所有的变化后，统一更新一次DOM，以上大概就是全部解析内容了。

### TODO React Fiber
这个还没学完

## 参考文献
[What is the Virtual DOM?「React官网」](https://reactjs.org/docs/faq-internals.html#what-is-the-virtual-dom)
[深入浅出React： 虚拟DOM Diff 算法解析](https://www.infoq.cn/article/react-dom-diff/)
链接挂掉了几个
深度剖析：如何实现一个 Virtual DOM 算法
