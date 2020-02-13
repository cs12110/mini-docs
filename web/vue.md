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
module.exports = { runtimeCompiler: true };
```

### 2.2 vue-router

官方文档:[link](https://router.vuejs.org/zh/installation.html)


