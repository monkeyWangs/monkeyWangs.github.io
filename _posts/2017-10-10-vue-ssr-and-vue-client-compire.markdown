---
layout: post
title: "实例PK ( Vue服务端渲染 VS 浏览器端渲染 )"
date: 2017-10-10 10:34:10 +0300
description: 本人在公司做 Vue 项目的时候，一直苦于产品、客户对首屏加载要求，SEO 的诉求，也想过很多解决方案，本次也是针对浏览器渲染不足之处，采用了服务端渲染，并且做了两个一样的 Demo作为比较... # Add post description (optional)
img:  vue-ssr.jpg
---

Vue 2.0 开始支持服务端渲染的功能，所以本文章也是基于 Vue 2.0 以上版本。网上对于服务端渲染的资料还是比较少，最经典的莫过于 Vue 作者尤雨溪大神的 Vue-hacker-news。本人在公司做 Vue 项目的时候，一直苦于产品、客户对首屏加载要求，SEO 的诉求，也想过很多解决方案，本次也是针对浏览器渲染不足之处，采用了服务端渲染，并且做了两个一样的 Demo作为比较，更能直观的对比 Vue 前后端的渲染。
话不多说，我们分别来看两个Demo:（欢迎star 欢迎pull request）
1. 浏览器端渲染[Demo][vue-client]
2. 服务端端渲染[Demo][vue-ssr]

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

#### 2.引入Vuex
Vuex 是一个专为 Vue.js 应用程序开发的状态管理模式。它采用集中式存储管理应用的所有组件的状态，并以相应的规则保证状态以一种可预测的方式发生变化。Vuex 也集成到 Vue 的官方调试工具 devtools extension，提供了诸如零配置的 time-travel 调试、状态快照导入导出等高级调试功能。
简而言之：Vuex 相当于某种意义上设置了读写权限的全局变量，将数据保存保存到该“全局变量”下，并通过一定的方法去读写数据。
Vuex 并不限制你的代码结构。但是，它规定了一些需要遵守的规则：

1. 应用层级的状态应该集中到单个 store 对象中。
2. 提交 mutation 是更改状态的唯一方法，并且这个过程是同步的。
3. 异步逻辑都应该封装到 action 里面。

对于大型应用我们会希望把 Vuex 相关代码分割到模块中。下面是项目结构示例：
``` bash
├── index.html
├── main.js
├── api
│   └── ... # 抽取出API请求
├── components
│   ├── App.vue
│   └── ...
└── store
    ├── index.js          # 我们组装模块并导出 store 的地方
    └── moving            # 电影模块
        ├── index.js      # 模块内组装，并导出模块的地方
        ├── actions.js    # 模块基本 action
        ├── getters.js    # 模块级别 getters
        ├── mutations.js  # 模块级别 mutations
        └── types.js      # 模块级别 types
```

所以我们开始在我们的src目录下新建一个名为 store 的文件夹 为了后期考虑 我们新建了moving 文件夹，用来组织电影，考虑到所有的 action,getters,mutations,都写在一起，文件太混乱，所以我又给他们分别提取出来。
stroe 文件夹建好，我们要开始在 main.js 里面引用 Vuex 实例：
``` javascript
import store from './store'
new Vue({
  el: '#app',
  router,
  store,
  template: '<App/>',
  components: { App }
})
```
这样，我们便可以在所有的子组件里通过 this.$store 来使用 Vuex 了。

#### 3.webpack proxyTable 代理跨域
webpack 开发环境可以使用 proxyTable 来代理跨域，生产环境的话可以根据各自的服务器进行配置代理跨域就行了。在我们的项目 config/index.js 文件下可以看到有一个 proxyTable 的属性，我们对其简单的改写
``` javascript
proxyTable: {
  '/api': {
    target: 'http://api.douban.com/v2',
    changeOrigin: true,
    pathRewrite: {
      '^/api': ''
    }
  }
}
```
这样当我们访问
``` localhost:8080/api/movie ```
的时候 其实我们访问的是
``` http://api.douban.com/v2/movie ```
这样便达到了一种跨域请求的方案。
至此，浏览器端的主要配置已经介绍完了，下面我们来看看运行的结果：

![浏览器端渲染]({{site.baseurl}}/assets/img/vue-ssr/client.png)

为了介绍浏览器渲染是怎么回事，我们运行一下 npm run build 看看我们的发布版本的文件，到底是什么鬼东西....
run build 后会都出一个dist目录 ，我们可以看到里面有个 index.html，这个便是我们最终页面将要展示的 html，我们打开，可以看到下面：

![浏览器端渲染html]({{site.baseurl}}/assets/img/vue-ssr/c-html.png)

观察好的小伙伴可以发现，我们并没有多余的 DOM 元素，就只有一个 div，那么页面要怎么呈现呢？答案是 js append，对，下面的那些js会负责 innerHTML。而js是由浏览器解释执行的，所以呢，我们称之为浏览器渲染，这有几个致命的缺点：
js 放在 DOM 结尾，如果 js 文件过大，那么必然造成页面阻塞。用户体验明显不好（这也是我我在公司反复被产品逼问的事情）
1. 不利于 SEO
2. 客户端运行在老的 JavaScript 引擎上
3. 对于世界上的一些地区人，可能只能用1998年产的电脑访问互联网的方式使用计算机。而 Vue 只能运行在 IE9 以上的浏览器，你可能也想为那些老式浏览器提供基础内容 - 或者是在命令行中使用 Lynx 的时髦的黑客

基于以上的一些问题，服务端渲染呼之欲出....

### 服务器端渲染豆瓣电影

先看一张 Vue 官网的服务端渲染示意图

![服务端渲染示意]({{site.baseurl}}/assets/img/vue-ssr/s-webpack.png)

从图上可以看出，SSR 有两个入口文件，client.js 和 server.js， 都包含了应用代码，webpack 通过两个入口文件分别打包成给服务端用的 server bundle 和给客户端用的 client bundle. 当服务器接收到了来自客户端的请求之后，会创建一个渲染器 bundleRenderer，这个 bundleRenderer 会读取上面生成的 server bundle 文件，并且执行它的代码， 然后发送一个生成好的 html 到浏览器，等到客户端加载了 client bundle 之后，会和服务端生成的DOM 进行 Hydration (判断这个 DOM 和自己即将生成的 DOM 是否相同，如果相同就将客户端的Vue实例挂载到这个 DOM 上， 否则会提示警告)。
具体实现：
我们需要 Vuex，需要 router，需要服务器，需要服务缓存，需要代理跨域....不急我们慢慢来。

#### 1.建立nodejs服务
首先我们需要一个服务器，那么对于 nodejs，express 是很好地选择。我们来建立一个server.js
``` javascript
const port = process.env.PORT || 8080
app.listen(port, () => {
  console.log(`server started at localhost:${port}`)
})
```
这里用来启动服务监听 8080 端口。
然后我们开始处理所有的 get 请求，当请求页面的时候，我们需要渲染页面

``` javascript
app.get('*', (req, res) => {
  if (!renderer) {
    return res.end('waiting for compilation... refresh in a moment.')
  }

  const s = Date.now()

  res.setHeader("Content-Type", "text/html")
  res.setHeader("Server", serverInfo)

  const errorHandler = err => {
    if (err && err.code === 404) {
      res.status(404).end('404 | Page Not Found')
    } else {
      // Render Error Page or Redirect
      res.status(500).end('500 | Internal Server Error')
      console.error(`error during render : ${req.url}`)
      console.error(err)
    }
  }

  renderer.renderToStream({ url: req.url })
    .on('error', errorHandler)
    .on('end', () => console.log(`whole request: ${Date.now() - s}ms`))
    .pipe(res)
})
```
然后我们需要代理请求，这样才能进行跨域，我们引入 http-proxy-middleware 模块：
``` javascript
const proxy = require('http-proxy-middleware');//引入代理中间件
/**
 * proxy middleware options
 * 代理跨域配置
 * @type {{target: string, changeOrigin: boolean, pathRewrite: {^/api: string}}}
 */
var options = {
  target: 'http://api.douban.com/v2', // target host
  changeOrigin: true,               // needed for virtual hosted sites
  pathRewrite: {
    '^/api': ''
  }
};

var exampleProxy = proxy(options);
app.use('/api', exampleProxy);
```

这样我们的服务端 server.js 便配置完成。接下来 我们需要配置服务端入口文件，还有客户端入口文件，首先来配置一下客户端文件，新建 src/entry-client.js

``` javascript
import 'es6-promise/auto'
import { app, store, router } from './app'

// prime the store with server-initialized state.
// the state is determined during SSR and inlined in the page markup.
if (window.__INITIAL_STATE__) {
  store.replaceState(window.__INITIAL_STATE__)
}

/**
 * 异步组件
 */
router.onReady(() => {
  // 开始挂载到dom上
  app.$mount('#app')
})

// service worker
if (process.env.NODE_ENV === 'production' && 'serviceWorker' in navigator) {
  navigator.serviceWorker.register('/service-worker.js')
}
```
客户端入口文件很简单，同步服务端发送过来的数据，然后把 Vue 实例挂载到服务端渲染的 DOM 上。
再配置一下服务端入口文件：src/entry-server.js
``` javascript
import { app, router, store } from './app'

const isDev = process.env.NODE_ENV !== 'production'

// This exported function will be called by `bundleRenderer`.
// This is where we perform data-prefetching to determine the
// state of our application before actually rendering it.
// Since data fetching is async, this function is expected to
// return a Promise that resolves to the app instance.
export default context => {
  const s = isDev && Date.now()

  return new Promise((resolve, reject) => {
    // set router's location
    router.push(context.url)

    // wait until router has resolved possible async hooks
    router.onReady(() => {
      const matchedComponents = router.getMatchedComponents()
      // no matched routes
      if (!matchedComponents.length) {
        reject({ code: 404 })
      }
      // Call preFetch hooks on components matched by the route.
      // A preFetch hook dispatches a store action and returns a Promise,
      // which is resolved when the action is complete and store state has been
      // updated.
      Promise.all(matchedComponents.map(component => {
        return component.preFetch && component.preFetch(store)
      })).then(() => {
        isDev && console.log(`data pre-fetch: ${Date.now() - s}ms`)
        // After all preFetch hooks are resolved, our store is now
        // filled with the state needed to render the app.
        // Expose the state on the render context, and let the request handler
        // inline the state in the HTML response. This allows the client-side
        // store to pick-up the server-side state without having to duplicate
        // the initial data fetching on the client.
        context.state = store.state
        resolve(app)
      }).catch(reject)
    })
  })
}
```

server.js 返回一个函数，该函数接受一个从服务端传递过来的 context 的参数，将 Vue 实例通过 promise 返回。context 一般包含 当前页面的url，首先我们调用 Vue-router 的 router.push(url) 切换到到对应的路由， 然后调用 getMatchedComponents 方法返回对应要渲染的组件， 这里会检查组件是否有 fetchServerData 方法，如果有就会执行它。
下面这行代码将服务端获取到的数据挂载到 context 对象上，后面会把这些数据直接发送到浏览器端与客户端的 Vue 实例进行数据(状态)同步。
``` javascript
context.state = store.state
```

然后我们分别配置客户端和服务端 webpack，这里可以在我的 github上fork下来参考配置，里面每一步都有注释，这里不再赘述。
接着我们需要创建app.js:
``` javascript
import Vue from 'vue'
import App from './App.vue'
import store from './store'
import router from './router'
import { sync } from 'vuex-router-sync'
import Element from 'element-ui'
Vue.use(Element)

// sync the router with the vuex store.
// this registers `store.state.route`
sync(store, router)

/**
 * 创建vue实例
 * 在这里注入 router  store 到所有的子组件
 * 这样就可以在任何地方使用 `this.$router` and `this.$store`
 * @type {Vue$2}
 */
const app = new Vue({
  router,
  store,
  render: h => h(App)
})

/**
 * 导出 router and store.
 * 在这里不需要挂载到app上。这里和浏览器渲染不一样
 */
export { app, router, store }
```
这样 服务端入口文件和客户端入口文件便有了一个公共实例 Vue, 和我们以前写的 Vue 实例差别不大，但是我们不会在这里将 app moun t到 DOM 上，因为这个实例也会在服务端去运行，这里直接将 app 暴露出去。
接下来创建路由 router，创建 Vuex 跟客户端都差不多。详细的可以参考我的项目...
到此，服务端渲染配置 就简单介绍完了，下面我们启动项目简单的看下

![服务端渲染结果]({{site.baseurl}}/assets/img/vue-ssr/ssr.png)

这里跟服务端界面一样，不一样的是url已经不是之前的 #/而变成了请求形式 /
这样每当浏览器发送一个页面的请求，会有服务器渲染出一个 DOM 字符串返回，直接在浏览器段显示，这样就避免了浏览器端渲染的很多问题。
说起 SSR，其实早在 SPA ( Single Page Application ) 出现之前，网页就是在服务端渲染的。服务器接收到客户端请求后，将数据和模板拼接成完整的页面响应到客户端。 客户端直接渲染， 此时用户希望浏览新的页面，就必须重复这个过程， 刷新页面. 这种体验在 Web 技术发展的当下是几乎不能被接受的，于是越来越多的技术方案涌现，力求 实现无页面刷新或者局部刷新来达到优秀的交互体验。但是 SEO 却是致命的，所以一切看应用场景，这里只为大家提供技术思路，为 Vue 开发提供多一种可能的方案。
为了更清晰的对比两次渲染的结果，我做了一次实验，把两个想的项目 build 后模拟生产环境，在浏览器 netWork 模拟网速 3g 环境，先来看看服务端渲染的结果：

![浏览器端渲染netWork]({{site.baseurl}}/assets/img/vue-ssr/c-network.png)

可以看到整体加载 DOM 一共花了832ms；用户可能在网络比较慢的情况下从远处访问网站 - 或者通过比较差的带宽。 这些情况下，尽量减少页面请求数量，来保证用户尽快看到基本的内容。

![服务端渲染netWork]({{site.baseurl}}/assets/img/vue-ssr/s-network.png)

然后我们可以看到其中有一个 vendor.js 达到了563KB，整体的加载时间达到了了8.19s，这是因为单页面文件的原因，会把所有的逻辑代码打包到一个js里面。可以用分 webpack 拆分代码避免强制用户下载整个单页面应用，但是，这样也远没有下载个单独的预先渲染过的 HTML文件性能高。




[vue-client]: https://github.com/monkeyWangs/doubanMovie
[vue-ssr]: https://github.com/monkeyWangs/doubanMovie-SSR