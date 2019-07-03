---
title: GMTC大会记
date: 2019-06-27 12:28:29
tags: 前端大会
---

# 急速秒卡0.3s完成渲染
## 闪开优化
Data PreFetch

## 容器选择
不用native，成本太高
Weex
H5 优化
PWA方向
SSR方向

## 重新理解页面渲染
数据模板
最快的是从内存中直接取出模板并且渲染；

## NSR预渲染
Native Server Render

## 移动端要求

## 微服务化
造轮子
首屏pureJSX + 非首屏Preact

## pureJsx
是一个没有Virtualdom的，但是最小组件还保留的React实现；
只有1KB；

<!-- more -->

---

# Bilibili

网页最佳打开时间 不应超过2秒

上下游的维度来进行优化

## 秒开
原来stage0：
loadHtml css loag-reporter jquery comment bundle bussiness video
优化stage0：
load HTML Base CSS video log-reporter

参考：（逐帧分析youtube）

原来Stage1:
palyer.js   init Player

优化Stage1:
vedio.js player.js => init player

原来Stage2:
Auth playUrl

优化Stage2:
HTML have playUrl ？playUrl(node ssr) : Auth playurl

ps: 充分利用nodessr

优化stage3:
避免使用第一个预检请求 + MSE（MediaSourceExtensions） 初始化与拉流并行

优化入口处理：
1、通过分析用户点击分析，推荐视频播放地址预取；
2、局部SPA；
3、入口部分预取播放器资源；
4、热门视频头部缓存预取（与页同时输出）

优化难点：
1、长尾优化（可能是CDN的问题）；

产品优化：
新版播放页：播放器上移，资源优先级高

视频类型：MPEG-DASH
这个牛逼⤴️

## 弹幕体验
基于RequestAnimationFrame
- css3（逐帧添加）
- Canvas（逐帧绘制）

弹幕碰撞检测，因为没有固定轨道，

优化点：复用弹幕节点，避免DOM频繁创建与移除

### 蒙版弹幕
基于css3 mask-image属性的弹幕，配合人工智能学习的路径点制造出防遮挡的效果

优化蒙版：
1. 自动降帧
2. 大屏幕下没有办法

---

# 基于dom的幻灯片实现

- 编辑器实现思路
- 编辑器的技术选型
- 编辑器的功能难点与解决方法

实现思路不同于模块化、组件化，是 从点到线，从线到面

office open xml 基于xml格式的office文档标准（用来定义某些元素到底需要实现什么东西）

## 选型考虑范围
1. 图形
- svg 有性能问题（1000以上会出现卡顿），但是渲染能力强，在移动端显示正常
- canvas 性能好，但是移动端（resize）的时候会出问题

最后选了svg
2. 富文本
HTML DOM contenteditable
第三方编辑器： Draft.js(react) \ CKEditor \ Quill \ ProseMisrror
- 最大化利用浏览器
- 数据与UI分离
- 保证数据与DOM的一致性
- 可扩展

## 实时协作算法
OT算法
用insert、remove和retain来表示文档内容及其修改

insert 插入
remove 移除
retain 保留

A用户 operate1 ⬇️ operate2' ->

B用户 operate2 ⤴️ operate1' ->

## 图形变换移动
- 移动
- 缩放
- 旋转

群体变换（多选中几个，做移动，做缩放，做旋转），如果固定坐标系，计算变更后的坐标，很麻烦，
因此，选择矩阵变换⬇️
**仿射变换**
P2(x', y') = T1 * T2 * T3 * P1(x, y)

---

# 数据可视化 React & D3
数据可视化分析，从数据中提取信息，用交互式图表展示出来，从而做出操作；

应用场景；
事件序列数据，帮助产品经理决策；
机器学习模型数据，帮助机器学习工程师分析与检测模型的更迭；

## 前端大数据可视化
出现的情况：高apis请求等
- 多样数据总结
- 多维度数据过滤
- 高频API请求
- 高度依赖 **React+ Redux**
- **Observable** 管理协调数据

## D3与数据可视化数据
有任务 + data + 概念模型 => 处理（合理利用人类视觉原理图形化数据） => 生成D3图表

人类视觉 不同的图形类型来处理数据
视觉变量 selective

### D3核心概念
数据驱动文档 => Data-Driven Documents
数据到Dom的映射

- 数据与DOM元素绑定
- 使用函数绘制到svg上

DOM 操作 - 与 React的异同
同： 数据与DOM一一对应
异：对于数据的变化直接掌控

### 辅助数据加工 - 数组变形
- 统计数据 => 数据总结
- 数据变形 => 数据分面

### 辅助数据加工 - 比例尺
主要是为了视觉元素支持

### 辅助数据加工 - 布局计算
### 数据图像的映射 - 颜色操作
### 数据图像的映射 - 路径生成
选用内置插值方式将数据转化为路径
Gestalt law of continuity
### 数据图像的映射 - 动画
利用插值函数，一行代码实现动画效果

## D3可视化的优势
- 数据与图形元素的一一对应
    - 用户与图形元素的交互 => 数据的选取
- 丰富的数据处理与图形映射功能
    - 辅助生成图形
    - 高度自由化

## 结合React
### D3的缺陷
- 代码冗长
    - Margin得放入函数计算
- 实现响应式图表需要直接处理window resize
- React与D3在DOM管理上存在潜在冲突

### 使用react-d3kit 实现D3代码复用
twitter团队搞得
d3kit：把D3代码封装成ES6

解决了margin枯燥的计算
解决了resize时候的问题

和react结合
- 用react 的状态来控制图表的重新渲染
- 利用Redux来统筹不同可视化组件的设置
- D3事件与react时间管理器的衔接

---

# 前端极致优化
选择性能优化工具
需要的能力（软实力-需求梳理、推广落地，硬实力-操作系统知识、数据处理能力、服务器知识、系统设计能力）
- 理解优化的收益和所处阶段
4个阶段
- 性能优化的数据支撑平台
现状、目标、成果
- 极致技术细节

ps:性能优化不是全部优化，在某些区间内优化

# 内容优化
1. 文本图片

文本压缩、图片压缩
（gzip、foundation）

图片 （svg => 可以gzip压缩）（无损压缩，压缩的是图片信息）（抽离像素通道）（图片转换成样式表）


