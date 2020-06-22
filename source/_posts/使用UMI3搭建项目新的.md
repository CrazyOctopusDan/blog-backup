---
title: 使用UMI3搭建项目心得
date: 2020-06-01 20:00:06
tags: UMI3 Dva Webpack
---

>  写这一篇之前感谢子杰同学的付出，给我们搭建了一套基于umi3的框架，内置了启动配置等等耗时的基础操作，达到了快速启动项目的目的。

-------

但是在使用中仍然需要注意几点，听我细细道来。

## UMI3
### 基础结构
这个是给小白说的，虽然umi3的写法超高度类似于reactjs的写法，但是不代表你就可以不去阅读umi3的文档了。

当然阅读了文档也不代表你就全部领悟了（可能因为作者默认你是老的开发者，所以有部分change他会说是从umi2中变换而来的，想了解还是得看一眼早期文档）

有意思的是他的项目目录结构：

`src/layouts `：作为页面的基础布局，可以有多个
`src/pages`: 作为各个路由对应的页面
这俩是默认的↑

`src/components`: 组件
`src/grpc`: 如果你接口的接入方式是grpc-web，注意这里面的目录层级结构需要和编译出来的proto文件所在的目录层级一致
`src/utils....`
这些是约定俗成的↑

### 项目配置
几乎全部在`.umirc.ts`文件内

不同于umi2x版本的层级配置，Umi3的配置全部拉平，第一次写就会有点懵逼，但是胆子大一点多尝试尝试就会好受的。

### 开始踩坑
#### 小坑1 routes 配置
routes 配置中要把路由结构写清晰（虽然他说可以默认按照文件目录结构生成，但是个人尝试了似乎并不好使）

什么意思呢？

比方说你拥有

```

pages/login

pages/center1

pages/center2

pages/center2/:hash

```

理论上通过路由，Umi3会帮你匹配到你目录结构下面的各个文件，然而，你的需求可能远不止此：

login页面的布局和其他页面并不相同
center2的详情页的和center2的页面路由很相似
...
这个时候如果你不在配置中写得明明白白的，我保证一个星期后你自己都不认得这个项目了。

正确示范↓

```JavaScript
routes: [
    {
        path: '/login',
        title: '登录',
        component: '@/pages/login',
    },
    {
        path: '/',
        component: '@/layouts/index',
        routes: [
            {
                path: '/center1',
                title: 'xx中心',
                component: '@/pages/center1',
            },
            {
                path: '/center2',
                title: 'yy中心',
                component: '@/pages/center2',
            },
            {
                exact: true,
                path: '/center2/:hash',
                title: 'yy详情',
                component: '@/pages/centers/[hash]',
             },
         ],
    }
]

```

这里有个坑存在：文档中默认 `exact: false`，但是如果你没事儿就千万不要自己写在route的配置中，写进去立马让这段匹配不好使！

其他的怎么定义路由就不细说了，文档里面全都有。

#### 小坑2 dva 类react redux
这个只需要在配置中写上即可，文档中一笔带过，不注意的人还发现不了嘿，不用自己再装一遍react-redux！

#### 小坑3 Umi3自带了antd4
这导致什么问题？（写作时间2020/5.20）我们现在的组件库仍然是antd3x

理论上我们开发项目要用到组件库，那就简单安装一下下laiye-antd即可，我们的laiye-antd会自己自觉引用项目中依赖的antd。

now注意！坑就在此，不知道发生了什么情况，在项目编译的时候会检查laiye-antd。由于 antd3x => antd4x中删除了moment组件，修改了select组件的样式，独立了icon等，导致了编译时会报错moment找不到！

百思不得其解，涉及到npm包和npm包里的npm包，情况相当不明朗，不过还是有解决方案的：

在umirc.ts文件中增加一行配置
```JavaScript
extraBabelPlugins: [
    ['import', {
        libraryName: 'laiye-antd',
        style: 'css'
    }, 'laiye-antd'],
],
```
声明，我们要按需加载laiye-antd，用什么加载什么，这样编译的时候就会放过moment组件了。

至于select组件使用时候丢失样式什么的，直接
```JavaScript
    import { select } from 'laiye-antd'

    import { select } from 'antd'
```
就行了

## 项目部署
### 概要
项目部署需要几个关键文件（夹）

/docker/baseimg.Dockerfile
/docker/Dockerfile
/docker/start.sh
/Jenkinsfile (由运维同学提供)
/pm2.json
在后端gitlab仓库中的配置模板项目（config-templates）中增加线上配置

### 具体步骤：
1. 得利于子杰同学的努力，项目中已经拥有了docker这个文件夹，只需要微调里面存放项目的地址即可；

2. 将自己的项目推到gitlab的仓库中，并且将gitlab项目地址、项目的各个环境的域名告诉运维同学；
3. 自己摸索到后端的仓库中，找到 config-templates 的项目，在对应的文件夹下面新（fu）建（zhi）好自己的配置文件，可以叫人帮你一起看看配置的对不对，然后发merge-request给运维同学；
4. 运维同学给到两样关键数据：（1.）Jenkins项目地址 （2）测试机上面的端口号；
5. **Jenkins地址**拿到后在自己的项目里面构建 baseimg 的 tag，只有拥有了baseimg才能走通master和testxx分支的构建；这里需要和运维同学一起联调；
6. 测试机上的端口号，将项目本地配置文件中的启动端口号更改成运维同学配置好的端口；通过ssh进入测试机，进入webroot目录中下载你的项目，走一遍yarn；yarn build；yarn pm2……;

第5步是部署测试环境、灰度环境、生产环境

第6步是部署联调环境

按照这个流程走下来基本上就么得问题了。
