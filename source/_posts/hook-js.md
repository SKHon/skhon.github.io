---
title: 谈谈js中的hook
date: 2020-01-03 03:25:24
categories: 前端
tags: [JavaScript]
comments: true
---

## 什么是 hook

hook 翻译过来就是钩子的意思。实质上是一种匹配的机制。可以先举个例子说明一下 hook 是什么。比如，前端和后端联调的时候，后端在 response 的时候，会给你返回各种状态码，你可以根据不同的状态码来分别进行前端的不同操作。这种逻辑，大多数同学会想到条件语句来实现:

```js
if (code === 200) {
    console.log("Ok");
} else if (code === 500) {
    console.log("Internal server error");
} else if (code === 404) {
    console.log("Not found");
}
```

当然也可以用 switch 来实现。那我们这里可以用 hook 来实现的话，就感觉代码很优雅，并且执行效率也相对快一些。我们可以把所有可能的 code 放在一个 list 中，code 作为 key，对应的内容作为 value，然后来进行匹配。代码如下：

```js
const codeList = {
    200: "Ok",
    404: "Not founnd",
    500: "Internal server error"
};
function print(input) {
    console.log(codeList[input]);
}
```

这样你输入对应的 key 值，那输出的就是对应的 value 值。
