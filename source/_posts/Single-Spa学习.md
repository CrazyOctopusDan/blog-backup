---
title: Single-Spa学习
date: 2019-07-15 11:17:13
tags:
---
# 触摸single-spa之道
## 什么是single-spa？
> single-spa是一个在前端应用程序中汇集多个javascript微内容的框架。 使用single-spa构建您的前端可带来许多好处，例如：

> * 在同一页面上使用多个框架而无需刷新页面（React，AngularJS，Angular，Ember或您正在使用的任何内容）
> * 独立部署微内容。
> * 使用新框架编写代码，而无需重写现有应用程序
> * 延迟加载代码，用于改善初始加载时间。

### 诞生概述
single-spa通过将生命周期应用于整个应用程序，从现代框架组件生命周期中获取灵感。 它的诞生起源于Canopy希望使用 `React + react-router` 而不是永远坚持使用我们的`AngularJS + ui-router`应用程序，现在single-spa支持几乎所有框架。 
    由于JavaScript因其众多框架的短命而臭名昭着，因此single-spa的设计者决定让开发者们想要的任何框架都很容易。
    
1. 应用程序，每个应用程序本身就是一个完整的SPA（有点）。 每个应用程序都可以响应url路由事件，并且必须知道如何从DOM引导，装载和卸载它们。 传统SPA和单一spa应用程序之间的主要区别在于它们必须能够与其他应用程序共存，并且它们不是每个都有自己的html页面。
 
 例如，您的React或Angular SPA是应用程序。 处于活动状态时，它们会侦听URL路由事件并将内容放在DOM上。 处于非活动状态时，它们不会侦听url路由事件，而是完全从DOM中删除。


2. 单个spa-config，它是html页面和使用single-spa注册应用程序的JavaScript。 每个应用程序都注册了三件事：

* 一个名字
* 加载应用程序代码的函数
* 用于确定应用程序何时处于活动/非活动状态的功能

### 很难使用吗？

#### 从技术栈架构上来说
single-spa适用于 ES5，ES6 +，TypeScript，Webpack，SystemJS，Gulp，Grunt，Bower，ember-cli或任何可用的构建系统。 开发者可以通过npm安装它，jspm安装它，或者如果愿意，甚至只需使用<script>标记。

作者的目标是尽可能简单地使用single-spa。 
但作者也指出，这是一种先进的架构，与前端应用程序的典型工作方式不同。

#### 从浏览器层面来说
single-spa适用于Chrome，Firefox，Safari，IE11和Edge。

## 储备知识
这里记录我在阅读了官网文档已经一些最佳实践之后归纳出的储备知识。
对于前端来说，要解决的问题一般分为两种阶段:
1. 开发阶段；
2. 部署阶段；

其中开发阶段要解决：
1. 第三方包的安装、使用、依赖的维护；
2. 自有代码的维护和使用；

### SystemJs
为什么要用 SystemJS ？
因为如果在项目中如果只使用单一规范，比如针对 AMD，我们可能会用 RequireJS；ES6 的模块，我们可能会用到 ES6 Module Loader Polyfill；CommonJS 规范的模块，我们可能用 SystemJS – 它同样可用于加载 AMD/ES6 模块。
systemjs 是一个最小系统加载工具，用来创建插件来处理可替代的场景加载过程，包括加载 CSS 场景和图片，主要运行在浏览器和 NodeJS 中。它是 ES6 浏览器加载程序的的扩展，将应用在本地浏览器中。通常创建的插件名称是模块本身，要是没有特意指定用途，则默认插件名是模块的扩展名称。

通常它支持创建的插件种类有：

``` Javascript
    # CSS 
    System.import('my/file.css!')
    
    # Image 
    System.import('some/image.png!image')
    
    # JSON 
    System.import('some/data.json!').then(function(json){})
    
    # Markdown 
    System.import('app/some/project/README.md!').then(function(html) {})
    
    # Text 
    System.import('some/text.txt!text').then(function(text) {})
    
    # WebFont 
    System.import('google Port Lligat Slab, Droid Sans !font')
```
### Webpack

