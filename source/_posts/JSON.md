---
title: 谈谈JSON.stringify
date: 2020-04-16 12:00:00
categories: 前端
tags: [js]
comments: true
copyright: true
---

## 先谈谈深拷贝的局限

有些文章介绍深拷贝的方法，说 JSON.parse(JSON.stringify(obj))可以进行深拷贝。其实这种深拷贝方法有很多局限：

<!--more-->

### 如果 json 里面有时间对象，则序列化结果：时间对象=>字符串的形式

```
let obj = {
    age: 18,
    date: new Date()
};
let objCopy = JSON.parse(JSON.stringify(obj));
console.log('obj', obj);
console.log('objCopy', objCopy);
console.log(typeof obj.date); // object
console.log(typeof objCopy.date); // string
```

运行结果：
![](/images/JSON_1.png)

### 如果 json 里有 RegExp、Error 对象，则序列化的结果将只得到空对象 RegExp、Error => {}

```
{
    let obj = {
        age: 18,
        reg: new RegExp('\\w+'),
        err: new Error('error message')
    };
    let objCopy = JSON.parse(JSON.stringify(obj));
    console.log('obj', obj);
    console.log('objCopy', objCopy);
}
```

运行结果：
![](/images/JSON_2.png)

### 如果 json 里有 function,undefined，则序列化的结果会把 function,undefined 丢失

```
{
    let obj = {
        age: 18,
        fn: function () {
            console.log('fn');
        },
        hh: undefined
    };
    let objCopy = JSON.parse(JSON.stringify(obj));
    console.log('obj', obj);
    console.log('objCopy', objCopy);
}
```

运行结果：
![](/images/JSON_3.png)

### 如果 json 里有 NaN、Infinity 和-Infinity，则序列化的结果会变成 null

```
{
    let obj = {
        age: 18,
        hh: NaN,
        isInfinite: 1.7976931348623157E+10308,
        minusInfinity: -1.7976931348623157E+10308
    };
    let objCopy = JSON.parse(JSON.stringify(obj));
    console.log('obj', obj);
    console.log('objCopy', objCopy);
}
```

运行结果：
![](/images/JSON_4.png)

### 如果 json 里有对象是由构造函数生成的，则序列化的结果会丢弃对象的 constructor

```
{
    function Person(name) {
        this.name = name;
    }
    let obj = {
        age: 18,
        p1: new Person('lxcan')
    };
    let objCopy = JSON.parse(JSON.stringify(obj));
    console.log('obj', obj);
    console.log('objCopy', objCopy);
    console.log(obj.p1.__proto__.constructor === Person); // true
    console.log(objCopy.p1.__proto__.constructor === Object); // true
}
```

运行结果：
![](/images/JSON_5.png)

### 如果对象中存在循环引用的情况也无法实现深拷贝

```
{
    let obj = {
        age: 18
    };
    obj.obj = obj;
    let objCopy = JSON.parse(JSON.stringify(obj));
    console.log('obj', obj);
    console.log('objCopy', objCopy);
}
```

运行结果：
![](/images/JSON_6.png)

## 再谈谈 JSON.stringify 的性能

在 hobber 框架里，打日志的时候，用了一个库，叫 flatted，在进行对象序列化的时候，用的这个库中的 stringify，flatted 序列化后的对象，是平铺的，key 值后面跟了一个序号，比如我有个对象：{ name: 'ljh', age: 18 }，其序列化之后长这样：[{"name":"1","age":18},"ljh"]，如果不知道的人就很懵逼，我要的结果是这样的呀：'{"name":"ljh","age":18}'。
其实当时的我也很懵逼，随手就想把序列化的方法改成 JSON.stringify。但是有个问题，就是 JSON.stringify 的性能。如果一条日志很大的时候，JSON.stringify 的性能堪忧。我们这里谈谈 JSON.stringify 的性能。

这里我写了一个统计解析时间的代码：

```
// 构造一个大对象
const fs = require('fs');
const { stringify } = require('flatted/cjs');
const fastJson = require('fast-json-stringify');
const stringify_fast = fastJson({
  title: 'Example Schema',
  type: 'object',
  patternProperties: {
    '^pro.*': {
      type: 'object',
      patternProperties: {
        '^person.*': {
          type: 'object',
          patternProperties: {
            '^name.*': {
              type: 'string'
            },
            '^age.*': {
              type: 'number'
            },
            '^sex.*': {
              type: 'string'
            }
          }
        },
        '^an.*': {
          type: 'object',
          patternProperties: {
            '^name.*': {
              type: 'string'
            },
            '^age.*': {
              type: 'number'
            },
            '^sex.*': {
              type: 'string'
            }
          }
        }
      }
    }
  }
});
function createBigObj(num) {
  let obj = {};
  for (let i = 0; i < num; i++) {
    let key = `pro${i}`;
    obj[key] = {
      [`person${i}`]: {
        [`name${i}`]: `name${i}`,
        [`age${i}`]: i,
        [`sex${i}`]: `sex${i}`
        // [`unknown${i}`]: function() {},
        // [`unknown1${i}`]: undefined,
        // [`unknown2${i}`]: NaN
      },
      [`an${i}`]: {
        [`name${i}`]: `name${i}`,
        [`age${i}`]: `${i + 100}`,
        [`sex${i}`]: `sex${i}`
      }
    };
  }
  return obj;
}
let newObj = createBigObj(10000);

function calTime(obj) {
  let oldTime = new Date().getTime();
  console.log(oldTime);
  //let work = JSON.stringify(obj);
  //let work = stringify(obj);

  let work = stringify_fast(obj);
  fs.writeFile('testfiles.txt', work, function() {
    console.log('finish');
    console.log('写文件时间：', new Date().getTime() - oldTime, 'ms');
  });
  let nowTime = new Date().getTime();
  console.log(nowTime);
  console.log('stringify计算时间', nowTime - oldTime, 'ms');
}
calTime(newObj);

```

循环 10000 次构造的大对象，对比如下：
原生的 JSON.stringify

```
1587026878274
1587026878288
stringify计算时间 14 ms
finish
写文件时间： 25 ms
```

flatted/cjs stringify

```
1587026906069
1587026906341
stringify计算时间 272 ms
finish
写文件时间： 286 ms
```

fast-json-stringify

```
1587026945311
1587026945523
stringify计算时间 212 ms
finish
写文件时间： 242 ms
```

使用命令看一下 testfiles.txt 文件大小：du -sh testfiles.txt

JSON.stringify 和 fast-json-stringify 生成的文件 1.4M，flatted/cjs stringify 生成的文件 1.9M，所以看统计结果原生的最快呀？？？什么情况下 fast-json-stringify 和 flatted/cjs 会比原生的 JSON.stringify 更快呢？？
