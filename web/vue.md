# vue

All about vue.

---

## 1. 安装 vue-cli

npm 更换阿里云地址

```sh
npm config set registry https://registry.npm.taobao.org
```

创建第一个 vue 项目

```sh
vue create hello-world
```

启动项目

```sh
cd hello-world
$ npm run serve
```

---

## 2. vue 组件

Vue 组件就像 Java 里面的 maven 引入依赖一样,需要的就 import 进来.

### 2.0 vue

使用脚手架来做,觉得好不习惯. 流下了没有技术的泪水.

Q: 如果要在页面加载的时候执行自己定义的方法,该怎么办?

A: follow me.

```js
// 4. 创建和挂载根实例。
new Vue({
  router,
  render: (h) => h(App),
  mounted: function () {
    // 自己定义需要加载的方法
    loadAuth();
  },
}).$mount("#app");
```

### 2.1 vue-router

tmd,一直卡在这一块,权限处理呀,卧槽.

官方文档:[link](https://router.vuejs.org/zh/installation.html)

进入前端项目所在路径,进行组件安装

```sh
# mr3306 @ mr3306 in ~/Box/projects/web/vue-shop on git:master x [17:31:29] C:130
$ pwd
/Users/mr3306/Box/projects/web/vue-shop

# mr3306 @ mr3306 in ~/Box/projects/web/vue-shop on git:master x [17:31:30]
$ npm install vue-router
```

项目引入组件

```js
import Vue from "vue";
import VueRouter from "vue-router";

Vue.use(VueRouter);
```

### 2.2 Axios

主要用于封装请求的组件,类似于 jQuery 里面的 ajax.

官网地址:[link](http://www.axios-js.com/zh-cn/docs/index.html)

进入前端项目所在路径,进行组件安装

```sh
# mr3306 @ mr3306 in ~/Box/projects/web/vue-shop on git:master x [17:26:10]
$ pwd
/Users/mr3306/Box/projects/web/vue-shop

# mr3306 @ mr3306 in ~/Box/projects/web/vue-shop on git:master x [17:27:35]
$  npm install axios

# 最新版本的vue-axios需要vue3.0以上,所以指定旧版本
# mr3306 @ mr3306 in ~/Box/projects/web/vue-shop on git:master x [17:27:35]
$  npm install --save vue-axios@2.1.0
```

引入 axios 组件

```js
import Vue from "vue";
import axios from "axios";
import VueAxios from "vue-axios";

Vue.use(VueAxios, axios);
```

### 2.3 ElementUI

一个挺好看的前端的 vue ui 组件,

官方文档:[link](https://element.eleme.cn/#/en-US/component/installation)

进入前端项目所在的路径,进行组件安装

```sh
# mr3306 @ mr3306 in ~/Box/projects/web/vue-shop on git:master o [17:25:54]
$ pwd
/Users/mr3306/Box/projects/web/vue-shop

# mr3306 @ mr3306 in ~/Box/projects/web/vue-shop on git:master o [17:25:58]
$ npm i element-ui -S
```

引入组件

```js
import Vue from "vue";
import ElementUI from "element-ui";
import "element-ui/lib/theme-chalk/index.css";
import App from "./App.vue";

Vue.use(ElementUI);

new Vue({
  el: "#app",
  render: (h) => h(App),
});
```

### 2.4 vuex

[vuex 官网地址 link](https://vuex.vuejs.org/zh/guide/actions.html)

Q: 你给我翻译,翻译,什么叫他妈的 vuex,什么叫 tmd vuex

A: Vuex 是一个专为 Vue.js 应用程序开发的状态管理模式,采用集中式存储管理应用的所有组件的状态.可简单理解为一个全局的状态管理器,我们可以把一些全局的状态存储在里面.当多个组件中显示这些状态时,只要在任意一个组件中改变这个状态,其余组件中的这个状态均会改变.

| 名称     | 概念                                                                                                                               | 备注 |
| -------- | ---------------------------------------------------------------------------------------------------------------------------------- | ---- |
| Store    | 相当于一个容器,它包含着应用中大部分的状态                                                                                          |
| State    | Store 中存储的状态,由于使用了单一状态树,即 Vuex 中存储的状态只存在一份,当这个状态发生改变时,和它绑定的组件中的这个状态均会发生改变 |
| Getter   | 从 State 中派生出的一些状态，可以认为是 State 的计算属性                                                                           |
| Mutation | 状态的变化,更改 Vuex 中的 State 的唯一方法是提交 Mutation                                                                          |
| Action   | 用于提交 Mutation 的动作,从而更改 Vuex 中的 State                                                                                  |
| Module   | Store 中的模块,由于使用单一状态树,应用的所有状态会集中到一个比较大的对象.为了解决以上问题,Vuex 允许我们将 Store 分割成模块         |

`mutations`示例:

```js
const store = new Vuex.Store({
  state: {
    count: 1,
  },
  mutations: {
    increment(state) {
      // 变更状态
      state.count++;
    },
  },
});
```

通过`commit`调用: `store.commit('increment')`

`actions`示例

```js
const store = new Vuex.Store({
  state: {
    count: 0,
  },
  mutations: {
    increment(state) {
      state.count++;
    },
  },
  actions: {
    increment(context) {
      context.commit("increment");
    },
  },
});
```

通过`dispatch`调用: `store.dispatch('increment')`

---

## 3. Issue

记录各种异常.

### 3.1 runtime 警告

在 vue-cli 3 里面有很多变态的东西 orz.

```
[Vue warn]: You are using the runtime-only build of Vue where the template compiler is not available. Either pre-compile the templates into render functions, or use the compiler-included build.
```

在根目录创建`vue.config.js`,里面填写如下内容:

```js
module.exports = {
  configureWebpack: {
    resolve: {
      alias: {
        vue$: "vue/dist/vue.esm.js",
      },
    },
  },
};
```

### 3.2 组件的使用

在 vue 里面,各种 this,这是何等的卧槽啊.

比如现在在项目里面引入了 ElementUI,但是想使用 ta 的消息组件:`message`,按照官网的

```js
this.$message("This is a message.");
```

这种 this 让我哭出了声.
