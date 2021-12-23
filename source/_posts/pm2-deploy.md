---
title: PM2 deploy
date: 2021-12-23 15:47:51
---

## 前言

[Nuxt.js](https://nuxtjs.org/) 是一个基于 Vue.js 的轻量级应用框架,可用来创建服务端渲染 (SSR) 应用，本篇来讲一下 **如何使用 PM2 部署 Nuxt 应用**。

## 创建 Nuxt 应用

对于如何创建 Nuxt 应用，这里不作赘述，这里建议直接参考 [官网](https://nuxtjs.org/docs/2.x/get-started/installation) 的做法，使用 `create-nuxt-app` 就好，可以参照 [这里](https://github.com/nuxt/create-nuxt-app/blob/master/README.md#features-tada) 的说明来了解创建应用时的配置选项。

创建成功后，查看 **package.json** 文件，发现 Nuxt 已内置多种预设命令：

```json
{
	"scripts": {
	    "dev": "nuxt",
	    "build": "nuxt build",
	    "start": "nuxt start",
	    "generate": "nuxt generate",
	    "lint:js": "eslint --ext \".js,.vue\" --ignore-path .gitignore .",
	    "lint:style": "stylelint \"**/*.{vue,css}\" --ignore-path .gitignore",
	    "prepare": "husky install",
	    "lint": "yarn lint:js && yarn lint:style"
	}
}
```

运行 `yarn build` 会打包 nuxt 应用并在根目录生成 `.nuxt` 目录，此时运行 `yarn start` 命令后访问 [http://localhost:3000/](http://localhost:3000/) 即可查看应用页面。

## PM2-进程守护

了解 NodeJS 的人都知道，通过 *CMD/PowerShell* 开启的 Node 服务，在关闭 *命令提示符* 窗口后，服务就断开了，页面也就访问不了。那么这如何解决呢？解决这个问题需要我们来了解一下 [PM2](https://pm2.keymetrics.io/docs/usage/quick-start/)：

> PM2 是一个守护进程管理器，它将帮助您管理和保持您的应用程序在线。

```powershell
npm install pm2@latest -g
# or
yarn global add pm2
```

运行上述命令完成 *pm2* 的安装，接下来在项目根目录创建一个 <font color='#0A7700'>*ecosystem.config.js*</font> 文件，内容如下：

```javascript
module.exports = {
  apps: [
    {
      name: 'NuxtAppName',
      exec_mode: 'cluster',
      instances: 'max', // Or a number of instances
      script: './node_modules/nuxt/bin/nuxt.js',
      args: 'start'
    }
  ]
}
```

好了，那么此时运行 `pm2 start` 后，你可以将任何 *命令提示符* 窗口关掉都可以访问 [http://localhost:3000/](http://localhost:3000/) 查看到应用页面，只要你不重启电脑。

## pm2-windows-startup-服务器重启后自动恢复 pm2 进程

依据上述内容，那么问题来了，电脑或服务器重启后，应用访问不了怎么办？实际的生产服务器中可能会有很多应用服务同时运行，总不能服务器宕机重启后一一手动敲命令重启服务吧。这里我们肯定想要在服务器重启后自动恢复应用服务（在 windows 系统中实践下），那么了解一下 `pm2-windows-startup` ：

##### 安装

```powershell
npm install pm2-windows-startup -g
# or
yarn global add pm2-windows-startup
```

##### 添加注册表项

```powershell
pm2-startup install
```

<font color=red>**注意：第一次安装或重启时可能会被杀毒软件警告，允许即可。**</font>

##### 保存当前的进程列表

在使用 `pm2 start` 启动服务后，使用 `pm2 save` 即可保存当前的进程列表。在重启电脑后访问 [http://localhost:3000/](http://localhost:3000/) 依然可以看到应用页面。

## 参考

- [Nuxt 官网 - 使用 PM2 部署 Nuxt](https://nuxtjs.org/docs/2.x/deployment/deployment-pm2) 
- [pm2-windows-startup - npm](https://www.npmjs.com/package/pm2-windows-startup) 
- [nuxt.js部署vue应用到服务端过程](https://segmentfault.com/a/1190000014450967) 
- [PM2 官网](https://pm2.keymetrics.io/) 

## 源码

更多细节参考 [源码](https://gitee.com/dizuncainiao/nuxt-deploy)
