---
title: 你见过js报的这些错吗
date: 2020-05-13 10:00:00
categories: 前端
tags: [node]
comments: true
copyright: true
---

## 数字--

在刷题过程中，这么写了一下代码：

```
var map = new Map()
map.set(1,1)
var result = map.get(1)--
```

会报这样的错：

```
Uncaught ReferenceError: Invalid left-hand side expression in postfix operation
```

【原因】数字不能--，得用变量
