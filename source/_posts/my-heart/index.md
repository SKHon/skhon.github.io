---
title: 最近的感想
date: 2021-07-16 00:00:00
categories: 杂谈
tags: [my-heart]
comments: true
copyright: true
---

## path的几个api用法
<!--more-->
```
const path = require('path')
console.log(`__dirname ==> ${__dirname}`)
console.log(`path.basename ==> ${path.basename(__dirname)}`)
console.log(`path.delimiter ==> ${path.delimiter}`)
console.log(`path.dirname ==> ${path.dirname(__dirname)}`)
console.log(`path.extname ==> ${path.extname('/user/test/index.jsx')}`)

// path.join([...paths]) 连接片段，并生成规范化路径
console.log(`path.join() ==> ${path.join('/base','test','ce','../')}`)

// path.relative(from, to) 生成相对路径
console.log(`path.relative() ==> ${path.relative('/root/test/ce','/root')}`)

// path.resolve([...paths]) 给定的路径序列从右到左进行处理，每个后续的 path 前置，直到构造出一个绝对路径
console.log(`path.resolve() ==> ${path.resolve()}`)
console.log(`path.resolve() ==> ${path.resolve('baidu','home')}`)

/*
输出：
__dirname ==> /Users/lianjia/Desktop/my notes/node/test
path.basename ==> test
path.delimiter ==> :
path.dirname ==> /Users/lianjia/Desktop/my notes/node
path.extname ==> .jsx
path.join() ==> /base/test/
path.relative() ==> ../..
path.resolve() ==> /Users/lianjia/Desktop/my notes/node/test
path.resolve() ==> /Users/lianjia/Desktop/my notes/node/test/baidu/home
*/
```
