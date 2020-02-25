---
title: 单元测试
date: 2020-01-14 03:25:24
categories: 前端
tags: [node]
comments: true
copyright: true
---

## 什么是单元测试

单元测试也叫做模块测试，就是对一个一个模块，来进行正确性检验对测试工作。如果接触测试的工作，测试人员在测你的项目之前都会写一些测试用例，然后这些测试用例可以覆盖到各种情况，然后用这些测试用例再一一检测你的程序，这是测<!--more-->试功能做的一部分。我们项目做单元测试也就是为了我们的程序能够尽可能的少出 bug。

## node 的单元测试

单元测试的框架很多，可以看一下各种单元测试框架的对比：
![](/images/test01.png)
mocha 用的最多，我用的也是 mocha，但是 mocha 没有断言库，那配一个 chai。

## demo

1. 先安装 mocha，chai

```
npm i --save-dev mocha
npm i --save-dev chai
```

2. 写 demo

```js
// modules/add.js
module.exports = function(a, b) {
    return a + b;
};

// test/add.spec.js
const { expect } = require("chai");
const add = require("../modules/add");
describe("测试加法模块", function() {
    it("1加2,应该为3", function() {
        expect(add(1, 2)).to.equal(3);
        //assert.equal(-1, [1, 2, 3].indexOf(-1));
    });
});
```

执行：

```
npx mocha add.spec.js
```

![](/images/test02.png)
