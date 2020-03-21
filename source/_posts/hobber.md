---
title: 整理一下hobber
date: 2020-03-04 00:00:00
categories: 前端
tags: [node]
comments: true
copyright: true
---

## 安装 hobber

1. 全局安装 hobber-cli

```
$ npm install @ke/hobber-cli -g --registry=http://registry.npmjs.lianjia.com:7001
```
<!--more-->
> note: hobber-cli 官网：http://registry.npmjs.lianjia.com/package/@ke/hobber-cli

2. 利用 hobber-cli 创建一个 hobber 项目

```
$ hobber new hobber-test
```

3. 安装依赖

```
$ cd hobber-test
$ npm i --registry=http://registry.npmjs.lianjia.com:7001
```

4. 本地启动

```
$ npm run dev
```

## 调试 hobber

目前这里有两种调试方法：

1. 可以通过自己打日志 console

2. 可以配置 vscode，来进行调试

launch.json 配置如下：

```
{
  // 使用 IntelliSense 了解相关属性。
  // 悬停以查看现有属性的描述。
  // 欲了解更多信息，请访问: https://go.microsoft.com/fwlink/?linkid=830387
  "version": "0.2.0",
  "configurations": [
    {
      "type": "node",
      "request": "launch",
      "name": "debug",
      "program": "${workspaceFolder}/src/index.ts",
      "outFiles": ["${workspaceFolder}/**/*.js"],
    }
  ]
}

```

## hobber 使用

### action

- 创建一个 action

```
$ hobber action /project/list
```

- 配置是否需要登录

```
export default {

  // 是否页面需要登陆 默认需要登陆
  // 具体逻辑在 configs/action 的 beforeHandler 中体现
  needLogin: false,

  async handler(ctx: Context) {

  },

} as ActionConfig
```
