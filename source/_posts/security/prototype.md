---
title: 【安全防御】原型链攻击
categories: 安全
tags: [Security]
comments: true
copyright: true
toc: true
---

安全问题是公司非常重视的问题，但是在我面试过程中，很多候选人只知道xss、csrf这两种，因为多数面经中，只会提到这两种。可以看到，前端工程师在平时的开发中，还是很少考虑安全问题的。由于本人在公司负责工程化建设相关工作，会涉及到项目的安全漏洞检测，所以后续多写一些相关的文章。
今天聊的是原型链污染攻击问题，我们先以一个非常简单的程序入手，我们知道在JavaScript中，一个对象有一个__proto__属性，它是指向Object.prototype的，像这样：
```
let obj = {};
console.log(obj.__proto__ === Object.prototype); // true
```

有了这个前置知识后，大家可以看下面的程序会输出什么。
```
let obj = {};
obj.__proto__.name = 'obj';
console.log(obj.name); // 这个很明显，是obj

let newObj = {};
console.log(newObj.name); // ???

```
执行后，你会发现，newObj.name也为obj！什么情况，我新定义的newObj对象，居然可以输出name属性？？没错，这就是原型链污染。**原因就是：改obj.__proto__,其实就是改了Object.prototype,而newObj的__proto__也是指向Object.prototype，这样Object的原型对象被偷偷改了，导致后面的对象不知道，这就是原型链污染的实质**

这种漏洞一般会出现在类似merge操作中，而很多工具库就提供merge方法，之前lodash就存在过原型链漏洞问题，后来修复了，如果存在漏洞，有些人就会故意往原型链上merge一些具有危险的属性，给系统带来危机。我们可以看一个简单的merge函数：
```
function merge(target, source) {
  for (let key in source) {
    if (key in source && key in target) {
      merge(target[key], source[key])
    } else {
      target[key] = source[key]
    }
  }
}
```
我们来验证一下：
```
let o1 = {}
let o2 = {a: 1, "__proto__": {b: 2}}
merge(o1, o2)
console.log(o1.a, o1.b) // 1 2

o3 = {}
console.log(o3.b) // undefined
```
我们可以看到这么搞，是没有污染的，原因是o2这么定义，__proto__会直接放在原型链上，不会当作o2的属性，可以这么理解：
o2.__proto__ ==> { b: 2, __proto__: Object.prototype}，而for in是能够遍历到原型链上的属性，所以会直接在o1中增加属性b为2。加入这么写会是啥样，看代码：
```
let o1 = {}
let o2 = JSON.parse('{"a": 1, "__proto__": {"b": 2}}')
merge(o1, o2)
console.log(o1.a, o1.b) // 1 2

o3 = {}
console.log(o3.b) // 2
```
这么写就污染了，因为这么写，__proto__会当作o2的一个属性，就是这样的：
```
o2 = {
  a: 1,
  __proto__: {
    b: 2
  }
}
```
这么会让o1.__proto__ = {b: 2},而o1的__proto__就是Object.prototype，随意就会导致Object的原型对象被悄悄的改了。

如何缓解原型链漏洞呢？这里提供了几种方式可参考：
> Object.freeze 将缓解几乎所有情况。冻结 Object 阻止添加新的 Prototype。
> 使用模式验证确保 JSON 数据包含预期属性，从而删除 JSON 中出现的 _proto_。
> 使用映射原语。它在 EcmaScript6 标准中引入，目前在 NodeJS 环境中备受支持。
> 使用 Object.create(null) 函数创建的Objects 不具有 _proto_ 属性。