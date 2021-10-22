# Team-web（前台部分）

## 创建项目

### vite创建初始项目

主要采用vite去构建项目。

```shell
npm init @vitejs/app
```

然后跟着提示完成即可！然后就会得到一个简单的目录文件。

![image-20211016202254187](https://cdn.jsdelivr.net/gh/baici1/image-host/newimg/20211016202301.png)

和常用的vue项目就缺少了很多插件配置，需要构建一个完整的vue项目，供后续项目进行开发。

### 添加相关插件与进行相关配置

为了减少负载（项目加载的文件），让文件运动更快，引入插件或者组件库，都选择按需引入方式。

> 1. 必备插件

* vue-router

```shell
npm install vue-router@next
```

* vuex

```shell
npm install vuex@next
```

* axios

```shell
npm install axios
```

* mock（看需求，此项目不需要）

```shell
npm install mockjs
```

* sass

```shell
npm install -D sass
```

> 2. 建立文件夹

```shell
team-web
├─ .gitignore
├─ .vscode
│  └─ extensions.json
├─ index.html
├─ package-lock.json
├─ package.json
├─ public
│  └─ favicon.ico
├─ README.md
├─ src
│  ├─ App.vue #入口文件
│  ├─ assets #静态文件
│  │  └─ logo.png 
│  ├─ components  #公共组件
│  │  └─ HelloWorld.vue
│  ├─ main.js  #主函数
│  ├─ router #路由
│  │  ├─ index.js
│  │  └─ modules #路由包（分组存放路由）
│  ├─ store #状态管理vuex
│  │  ├─ index.js #主文件
│  │  └─ modules #分组管理数据
│  ├─ utils #工具文件
│  └─ views #页面
└─ vite.config.js #项目配置文件
```

> 3. 进行项目代码配置

* router

学习地址：

https://www.jianshu.com/p/c805b74e1f14?utm_campaign

[https://router.vuejs.org/zh/guide/advanced/scroll-behavior.html#%E5%BC%82%E6%AD%A5%E6%BB%9A%E5%8A%A8](https://router.vuejs.org/zh/guide/advanced/scroll-behavior.html#异步滚动)

```javascript
import { createRouter, createWebHashHistory } from "vue-router";
/* 菜单栏的路由 */
// 固定菜单
export const fixedRoutes = [];
// 动态菜单
export const asyncRoutes = [];

const router = createRouter({
  history: createWebHashHistory(),
  routes: [
    { path: "/", redirect: "/home" },
    // ...login,
    // //...redirect, // 统一的重定向配置
    // ...asyncRoutes,
  ],
  scrollBehavior(to, from, savedPosition) {
    // savedPosition 会在你使用浏览器前进或后退按钮时候生效
    // 这个跟你使用 router.go() 或 router.back() 效果一致
    //这里主要处理当你的home滚动再底部，跳转页面也是底部的bug情况
    //主要是让页面回到顶部
    if (savedPosition) {
      return savedPosition;
    } else {
      return { x: 0, y: 0 };
    }
  },
});
export default router;
```

* store

学习博客地址：

https://www.cnblogs.com/smallyi/p/14255374.html

https://cn.vitejs.dev/guide/features.html#glob-import

```javascript
import { createStore } from "vuex";
//知识点1： 可用于模块的批量导入，类同于import引入同一文件夹下多个文件。
//支持使用特殊的 import.meta.glob 函数从文件系统导入多个模块，
const modulesFiles = import.meta.globEager("./modules/*.js");
// 知识点2：reduce(()=> {total, currentValue, currentIndex, arr}, initValue)
// 参数： 1. 执行每条数据的函数 2. 传递给函数的初始值，可选（以前没发现初始值的妙用-可用于统计个数、去重等）
// 函数的参数
// 1. total 必需。初始值, 或者计算结束后的返回值。如果设置初始值就用初始值，否则就是函数的第一条数据
// 2. currentValue 必需。当前元素
const modules = Object.entries(modulesFiles).reduce((modules, [path, mod]) => {
  const moduleName = path.replace(/^\.\/modules\/(.*)\.\w+$/, "$1");
  modules[moduleName] = mod.default;
  return modules;
}, {});

export default createStore({
  modules,
});

```

* 配置vite

```javascript
import { defineConfig } from "vite";
import vue from "@vitejs/plugin-vue";
import path from "path";
// https://vitejs.dev/config/
export default defineConfig({
  plugins: [vue()],
  proxy: {
    //配置跨域请求
    "/api": {
      target: "http://localhost:8199",
      changeOrigin: true,
      rewrite: (path) => path.replace(/^\/api/, ""),
    },
  },
  resolve: {
    //设置@的代表的位置
    alias: {
      "/@/": path.resolve(__dirname, "./src"),
    },
  },
  //配置别名
});

```

main.vue

```vue
<template>
  <div id="app">
    <router-view />
  </div>
</template>

<style lang="scss">
html,
body,
#app {
  width: 100%;
  height: 100%;
  margin: 0;
  padding: 0;
}
</style>

```

main.js

```javascript
import { createApp } from "vue";
import App from "./App.vue";
// 引入store
import store from "./store";
//引入路由
import router from "./router";
createApp(App).use(store).use(router).mount("#app");
```

到这里，基本上你有一个完整的，可以进行开发的，空白的vue项目了。

### 选择常用的组件库

主打组件库：Ant Design of Vue 

> 1. 安装组件库

```shell
$ npm install ant-design-vue@next --save
```

> 2. 入口文件引入组件库

```js
import { DatePicker } from "ant-design-vue";
//引入样式
import "ant-design-vue/dist/antd.css"; // or 'ant-design-vue/dist/antd.less'
app.use(DatePicker);
```

> 3. 按需加载

插件：https://github.com/antfu/unplugin-vue-components

```javascript
// vite.config.js
import ViteComponents, { AntDesignVueResolver } from 'vite-plugin-components';

export default {
  plugins: [
    /* ... */
    ViteComponents({
      customComponentResolvers: [AntDesignVueResolver()],
    }),
  ],
};
```

基本上到这里你就可以开始搭建vue项目。初始化的工作全部完成！！

## 搭建登录注册页面

之前比较喜欢米修在线的一个项目的登录注册页面。后面是他的页面对接到我的项目里面。

[米修在线](https://www.bilibili.com/video/BV1PZ4y1G7bu?from=search&seid=6642265194104581207&spm_id_from=333.337.0.0)

```vue
<template>
  <div class="container" :class="{ 'sign-up-mode': signUpMode }">
    <div class="form-warp">
      <form class="sign-in-form">
        <h2 class="form-title">登录</h2>
        <input placeholder="用户名" />
        <input type="password" placeholder="密码" />
        <div class="submit-btn">立即登录</div>
      </form>
      <form class="sign-up-form">
        <h2 class="form-title">注册</h2>
        <input placeholder="用户名" />
        <input type="password" placeholder="密码" />
        <div class="submit-btn">立即注册</div>
      </form>
    </div>
    <div class="desc-warp">
      <div class="desc-warp-item sign-up-desc">
        <div class="content">
          <button id="sign-up-btn" @click="signUpMode = !signUpMode">
            注册
          </button>
        </div>
        <img src="@/assets/svg/log.svg" alt="" />
      </div>
      <div class="desc-warp-item sign-in-desc">
        <div class="content">
          <button id="sign-in-btn" @click="signUpMode = !signUpMode">
            登录
          </button>
        </div>
        <img src="@/assets/svg/register.svg" alt="" />
      </div>
    </div>
  </div>
</template>

<script>
import { ref } from "@vue/reactivity";
export default {
  name: "Login",
  setup() {
    const signUpMode = ref(false);
    return {
      signUpMode,
    };
  },
};
</script>

<style lang="sass" scoped src="@/assets/css/login.scss"></style>


```

相关css就不贴了。



## 错误记录

### 错误1`Uncaught SyntaxError: The requested module '/node_modules/.vite/vuex.js?v=84828812' does not provide an export named 'createStore'`

![image-20211017115322859](https://cdn.jsdelivr.net/gh/baici1/image-host/newimg/20211017115329.png)

如果查询他的错误来源，你就会发现。

![image-20211017115416732](https://cdn.jsdelivr.net/gh/baici1/image-host/newimg/20211017115416.png)

其实这个问题我并没有得到解决，有一部分的博客和issue说过，但是我没有解决成功。

参考博客：

https://ostack.cn/?qa=172770/

https://github.com/vitejs/vite/issues/630

我的解决方式：

> 重新创建vue初始化项目

参考博客：

https://juejin.cn/post/7001857870035681316



### 注意点一：vue页面引入外部css

```vue
<style lang="scss" scoped src="@/assets/css/login.css">
</style>
```

参考文章：https://segmentfault.com/a/1190000012728854

**但是这里会出现一个问题。你当前页面写的css样式都会不生效！**

### 注意点二：表单验证名字与绑定字段要一致

![image-20211019153945036](https://cdn.jsdelivr.net/gh/baici1/image-host/newimg/20211019153945.png)

否则你会出现以下现象

此时输入有字符了，却还是提醒没有输入情况。

![image-20211019154031883](https://cdn.jsdelivr.net/gh/baici1/image-host/newimg/20211019154031.png)
