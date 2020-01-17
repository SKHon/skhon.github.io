---
title: node调试
date: 2020-01-09 03:25:24
categories: 前端
tags: [node]
comments: true
copyright: true
---

## node 调试的几种方法

在看 hobber 源码的时候，想看下一个请求是如何在 hobber 中跑的，我开始根据自己的理解以及猜测，是在 node_modules 中的包里打 console.log，这种方式感觉太累了，并且对整个 hobber 框架的理解效率不是很高，于是我问了一下 hobber 作者（黄老师）<!--more-->,黄老师给我的答复是，他当时写 hobber 的时候，调试都是 console。。。好吧，我感觉我作为一个刚刚接触 hobber 的新手来说，光 console 好像救不了我。于是就研究了一下 node 的调试方式。

### vscode

网上有很多相关调试 node 的文章，但是笔者感觉最好还是看官网，我认为学习任何一门技术，首要看的资料是官网，因为那个是最权威的。我这里记录一下我用 vscode 调试 node 的一种方式。

配置一下 launch.json 文件即可：

```js
{
    // 使用 IntelliSense 了解相关属性。
    // 悬停以查看现有属性的描述。
    // 欲了解更多信息，请访问: https://go.microsoft.com/fwlink/?linkid=830387
    "version": "0.2.0",
    "configurations": [
        {
            "type": "node",
            "request": "launch",
            "name": "hobber-debug",
            "program": "${workspaceFolder}/server/src/index.ts",
            // "preLaunchTask": "tsc: build - tsconfig.json",
            "outFiles": [
                "${workspaceFolder}/server/dist/*.js"
            ]
        }
    ]
}
```

这里贴一下 vscode debug 的官方介绍：[node debug in vscode](https://code.visualstudio.com/docs/nodejs/nodejs-debugging)

### inspector

这个 node 自带的就不要考虑了，太麻烦了。

### Chrome(55+) 开发工具

> 版本：
> Node.js 6.3+
> Chrome 55+

Chrome 打开：chrome://inspect
启动 node：

```js
node --inspect-brk app.js  // 在程序开头停止
node --inspect app.js      // 不会在程序开头停止
```
