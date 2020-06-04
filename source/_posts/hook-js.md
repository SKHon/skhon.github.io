---
title: 谈谈hook
date: 2019-11-03 03:25:24
categories: 前端
tags: [JavaScript]
comments: true
copyright: true
---

## 什么是 hook

概括的说：hook 是将一系列要执行的动作注册到一个统一的入口，程序通过调用 hook 来执行注册的动作。而一般执行 hook 的常规做法（并不是所有的 hook 都是这么调用的）是通过匹配的方式来执行。<!--more-->
我们可以先从匹配机制说起，先举个简单的例子。比如，前端和后端联调的时候，后端在 response 的时候，会给你返回各种状态码，你可以根据不同的状态码来分别进行前端的不同操作。这种逻辑，大多数同学会想到条件语句来实现:

```js
if (code === 200) {
  console.log('Ok');
} else if (code === 500) {
  console.log('Internal server error');
} else if (code === 404) {
  console.log('Not found');
}
```

当然也可以用 switch 来实现。那我们这里可以用 hook 来实现的话，就感觉代码很优雅，并且执行效率也相对快一些。我们可以把所有可能的 code 放在一个 list 中，code 作为 key，对应的内容作为 value，然后来进行匹配。代码如下：

```js
const codeList = {
  200: function() {
    console.log('Ok');
  },
  404: function() {
    console.log('Not founnd');
  },
  500: function() {
    console.log('Internal server error');
  }
};
function print(input) {
  codeList[input]();
}
print(200);
```

上述程序运行结果为："Ok"，那输出 Ok 的这个操作就是一个 hook，我们一共有三个 hook，而 print 为一个主流程，我们为这个主流程里面插入的我们注册的操作，就是 hook。再来一张丑图：
![hook 图解](/images/hook01.png)
start->end 是一个完整的流程，我们需要在这个流程里面插入我们想要自己实现的功能，就可以将这些功能先注册到一个数组中，然后调用的时候，可以 Array[key]这种方式，执行我们的 hook。

## js 中的函数劫持

劫持的意思，大家都懂，在程序中的劫持，可以理解在一个常规的程序流程中，可以插入一些我们自己想执行的一些操作，进而改变整个过程。举个简单的例子，比如，我想劫持 alert，可以这么做：

```js
var _alert = alert;
window.alert = function(s) {
  console.log('Hooked!');
  _alert(s);
};
alert('hello world');
```

这个过程可以理解为：先保存原来的 alert，然后重写 alert，这样在执行 alert 的时候，就是执行的重写之后的。那么重写的过程我们可以不暴露出来，那么对于用 alert 的用户来说，好像和以前的执行结果不太一样了。这就是`alert 函数被劫持了`。
我们可以再联想一下 hook 思想，hook 就是在一个完整流程中，插入我们想要实现的功能，那么原本的 alert 就是一个完整的执行流程，我们执行它，就是弹一个对话框，但是上面的例子展示了，在 alert 中添加了一条 console.log("Hooked!")这样的语句，这就是 hook 的思想。
