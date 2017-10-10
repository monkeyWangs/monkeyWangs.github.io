---
layout: post
title: "实例PK ( Vue服务端渲染 VS 浏览器端渲染 )"
date: 2017-10-10 10:34:10 +0300
description: Vue 2.0 开始支持服务端渲染的功能，所以本文章也是基于 Vue 2.0 以上版本。网上对于服务端渲染的资料还是比较少，最经典的莫过于 Vue 作者尤雨溪大神的 Vue-hacker-news。本人在公司做 Vue 项目的时候，一直苦于产品、客户对首屏加载要求，SEO 的诉求，也想过很多解决方案，本次也是针对浏览器渲染不足之处，采用了服务端渲染，并且做了两个一样的 Demo作为比较，更能直观的对比 Vue 前后端的渲染。 # Add post description (optional)
img:  vue-ssr.jpg
---
Vue 2.0 开始支持服务端渲染的功能，所以本文章也是基于 Vue 2.0 以上版本。网上对于服务端渲染的资料还是比较少，最经典的莫过于 Vue 作者尤雨溪大神的 Vue-hacker-news。本人在公司做 Vue 项目的时候，一直苦于产品、客户对首屏加载要求，SEO 的诉求，也想过很多解决方案，本次也是针对浏览器渲染不足之处，采用了服务端渲染，并且做了两个一样的 Demo作为比较，更能直观的对比 Vue 前后端的渲染。
话不多说，我们分别来看两个Demo:（欢迎star 欢迎pull request）
1. 浏览器端渲染[Demo][https://github.com/monkeyWangs/doubanMovie]
2. 服务端端渲染[Demo][https://github.com/monkeyWangs/doubanMovie-SSR]
两套代码运行结果都是为了展示豆瓣电影的，运行效果也都是差不多，下面我们来分别简单的阐述一下项目的机理：
###  浏览器端渲染豆瓣电影
首先我们用官网的脚手架搭建起来一个 Vue 项目：
``` bash
npm install -g vue-cli
vue init webpack doubanMovie
cd doubanMovie
npm install
npm run dev
```
这样便可以简单地打起来一个cli框架，下面我们要做的事情就是分别配置 Vue-router, Vuex,然后配置我们的 webpack proxyTable 让他支持代理访问豆瓣API。

#### 1.配置Vue-router
我们需要三个导航页：正在上映、即将上映、Top250；一个详情页，一个搜索页。这里我给他们分别配置了各自的路由。在 router/index.js 下配置以下信息：

``` javascript
import Vue from 'vue'
import Router from 'vue-router'
import Moving from '@/components/moving'
import Upcoming from '@/components/upcoming'
import Top250 from '@/components/top250'
import MoviesDetail from '@/components/common/moviesDetail'

import Search from '@/components/searchList'

Vue.use(Router)
/**
 * 路由信息配置
 */
export default new Router({
  routes: [
    {
      path: '/',
      name: 'Moving',
      component: Moving
    },
    {
      path: '/upcoming',
      name: 'upcoming',
      component: Upcoming
    },
    {
      path: '/top250',
      name: 'Top250',
      component: Top250
    },
    {
      path: '/search',
      name: 'Search',
      component: Search
    },
    {
      path: '/moviesDetail',
      name: 'moviesDetail',
      component: MoviesDetail
    }

  ]
})
```
这样我们的路由信息配置好了，然后每次切换路由的时候，尽量避免不要重复请求数据，所以我们还需要配置一下组件的 keep-alive：在 app.vue 组件里面。
``` html
<keep-alive exclude="moviesDetail">
   <router-view></router-view>
</keep-alive>
```
这样一个基本的 Vue-router 就配置好了。