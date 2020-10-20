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

## 2. vue

### 2.1 issue

#### runtime 警告

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

### 2.2 vue-router

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

### 2.3 ElementUI

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

### 2.4 Axios

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
