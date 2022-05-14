---
title: 【规范建设】为代码自动生成文档
categories: 工程化建设
tags: [Engineering]
comments: true
copyright: true
toc: true
---

[JSDoc](https://github.com/jsdoc/jsdoc) 是一个自动化生成 JavaScript 文档工具，它是利用对 JavaScript 函数的特定注释来编译成 HTML 文件的一个文档工具。

### 安装
```
npm install jsdoc -g
npm install jsdoc -save-dev
```

### 使用

只要在 JavaScript  中写好注释，利用命令即可：
```
jsdoc a.js b.js ...
```
复制代码当然我们也可以在项目下定义 jsdoc.json 配置文件，通过 -c 参数来指定：
```
jsdoc -c jsdoc.json
```
复制代码可以在 package.json 中的 scripts 添加命令：
```
{
	"scripts": {
  	"docs": "jsdoc -c jsdoc.json"
  }
}
```
复制代码这样我们就可以通过在项目下执行 npm run docs 命令来生成文档了。

### 配置文件

常用的配置文件
```
{
  "source": {
    "include": [ "src/" ],
    "exclude": [ "src/libs" ]
  },
  "opts": {
    "template": "templates/default",
    "encoding": "utf8",
    "destination": "./docs/",
    "recurse": true,
    "verbose": false
  }
}
```

- <font color=red>source</font> 表示传递给 JSDOC 的文件
- <font color=red>source.include</font> 表示 JSDOC 需要扫描哪些文件
- <font color=red>source.exclude</font> 表示 JSDOC 需要排除哪些文件
- <font color=red>opts</font> 表示传递给 JSDOC 的选项
- <font color=red>opts.template</font> 生成文档的模板，默认是 templates/default
- <font color=red>opts.encoding</font> 读取文件的编码，默认是 utf8
- <font color=red>opts.destination</font> 生成文档的路径，默认是 ./out/
- <font color=red>opts.recurse</font> 运行时是否递归子目录
- <font color=red>opts.verbose</font> 运行时是否输出详细信息，默认是 false

### 注释
```
/**
* @author Mondo
* @description list 数据结构 转换成 树结构
* @param {Array} data 需要转换的数据
* @param {String} id 节点 id
* @param {String} pid 父级节点 id
* @param {String} child 子树为节点对象的某个属性值
* @param {Object} labels 需要新增的字段名集合 { label: 'category_name' }
* @return {Array}
*
* @example
* formatListToTree({data: [{id:1}, {id: 2}, {id: 3, pid: 1}]})
* =>
* [ { id: 1, children: [ {id: 3, pid: 1} ] }, { id: 2 } ]
*/
```
常见的 JavaScript 块级注释，必须以 /** 开头，不然会被忽略掉。
下面介绍一些常见的级块标签：

- <font color=red>@author</font> 该类/方法的作者。
- <font color=red>@class</font>  表示这是一个类。
- <font color=red>@function/@method</font>  表示这是一个函数/方法(这是同义词)。
- <font color=red>@private</font> 表示该类/方法是私有的，JSDOC 不会为其生成文档。
- <font color=red>@name</font> 该类/方法的名字。
- <font color=red>@description</font> 该类/方法的描述。
- <font color=red>@param</font> 该类/方法的参数，可重复定义。
- <font color=red>@return</font> 该类/方法的返回类型。
- <font color=red>@link</font> 创建超链接，生成文档时可以为其链接到其他部分。
- <font color=red>@example</font> 创建例子。

这里是所有的[Block Tags](https://jsdoc.app/)

### 模板
JSDoc 默认的主题可能不近如人意，不过大型交友网站上给我们提供了还不错的主题，只要我们对应 install 下来配置就行。推荐两款还不错的主题：
- docdash
- minami

使用方式为：
1. 安装依赖
```
npm install docdash --save-dev
```
2. 配置
```
{
  "templates": {
    "cleverLinks": true,
    "default": {
      "layoutFile": "node_modules/docdash"
    }
  }
}
```

### 效果
<img src="/images/engineering/jsdoc.png" >