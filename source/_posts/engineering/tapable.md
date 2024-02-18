---
title: 【基础架构】构建插件机制的一把利器tapable
categories: 工程化建设
tags: [Architecture]
comments: true
copyright: true
toc: true
---

## tapable是什么
tapable是webpack的核心模块，也是webpack团队维护的，是webpack plugin的基本实现方式。他的主要功能是为使用者提供强大的hook机制，webpack plugin就是基于hook的。
主要API
下面是官方文档中列出来的主要API，所有API的名字都是以Hook结尾的：
```
const {
	SyncHook,
	SyncBailHook,
	SyncWaterfallHook,
	SyncLoopHook,
	AsyncParallelHook,
	AsyncParallelBailHook,
	AsyncSeriesHook,
	AsyncSeriesBailHook,
	AsyncSeriesWaterfallHook
 } = require("tapable");
```

这些API的名字其实就解释了他的作用，注意这些关键字：Sync, Async, Bail, Waterfall, Loop, Parallel, Series。下面分别来解释下这些关键字：
- Sync：这是一个同步的hook
- Async：这是一个异步的hook
- Bail：Bail在英文中的意思是保险，保障的意思，实现的效果是，当一个hook注册了多个回调方法，任意一个回调方法返回了不为undefined的值，就不再执行后面的回调方法了，就起到了一个“保险丝”的作用。
- Waterfall：Waterfall在英语中是瀑布的意思，在编程世界中表示顺序执行各种任务，在这里实现的效果是，当一个hook注册了多个回调方法，前一个回调执行完了才会执行下一个回调，而前一个回调的执行结果会作为参数传给下一个回调函数。
- Loop：Loop就是循环的意思，实现的效果是，当一个hook注册了回调方法，如果这个回调方法返回了true就重复循环这个回调，只有当这个回调返回undefined才执行下一个回调。
- Parallel：Parallel是并行的意思，有点类似于Promise.all，就是当一个hook注册了多个回调方法，这些回调同时开始并行执行。
- Series：Series就是串行的意思，就是当一个hook注册了多个回调方法，前一个执行完了才会执行下一个。
Parallel和Series的概念只存在于异步的hook中，因为同步hook全部是串行的。

## SyncHook
```
// 同步串行	不关心监听函数的返回值
const { SyncHook } = require("tapable");

// 实例化一个加速的hook
const accelerate = new SyncHook(["newSpeed"]);

// 注册第一个回调，加速时记录下当前速度
accelerate.tap("LoggerPlugin", (newSpeed) =>
  console.log("LoggerPlugin", `加速到${newSpeed}`)
);

// 再注册一个回调，用来检测是否超速
accelerate.tap("OverspeedPlugin", (newSpeed) => {
  if (newSpeed > 120) {
    console.log("OverspeedPlugin", "您已超速！！");
  }
});

// 再注册一个回调，用来检测速度是否快到损坏车子了
accelerate.tap("DamagePlugin", (newSpeed) => {
  if (newSpeed > 300) {
    console.log("DamagePlugin", "速度实在太快，车子快散架了。。。");
  }
});

// 触发一下加速事件，看看效果吧
accelerate.call(500);


```

## SyncBailHook
```
// 同步串行	只要监听函数中有一个函数的返回值不为 undefined，则跳过剩下所有的逻辑
const { SyncBailHook } = require("tapable");    // 使用的是SyncBailHook

// 实例化一个加速的hook
const accelerate = new SyncBailHook(["newSpeed"]);

accelerate.tap("LoggerPlugin", (newSpeed) =>
  console.log("LoggerPlugin", `加速到${newSpeed}`)
);

// 再注册一个回调，用来检测是否超速
// 如果超速就返回一个错误
accelerate.tap("OverspeedPlugin", (newSpeed) => {
  if (newSpeed > 120) {
    console.log("OverspeedPlugin", "您已超速！！");

    return new Error('您已超速！！');
  }
});

accelerate.tap("DamagePlugin", (newSpeed) => {
  if (newSpeed > 300) {
    console.log("DamagePlugin", "速度实在太快，车子快散架了。。。");
  }
});

accelerate.call(500);

```

## SyncLoopHook
```
// 同步循环	当监听函数被触发的时候，如果该监听函数返回true时则这个监听函数会反复执行，如果返回 undefined 则表示退出循环
const { SyncLoopHook } = require("tapable");

const accelerate = new SyncLoopHook(["newSpeed"]);

accelerate.tap("LoopPlugin", (newSpeed) => {
  console.log("LoopPlugin", `循环加速到${newSpeed}`);

  return new Date().getTime() % 5 !== 0 ? true : undefined;
});

accelerate.tap("LastPlugin", (newSpeed) => {
  console.log("循环加速总算结束了");
});

accelerate.call(100);

```

## SyncWaterfallHook
```
// 同步串行	上一个监听函数的返回值可以传给下一个监听函数
const { SyncWaterfallHook } = require("tapable");

const accelerate = new SyncWaterfallHook(["newSpeed"]);

accelerate.tap("LoggerPlugin", (newSpeed) => {
  console.log("LoggerPlugin", `加速到${newSpeed}`);

  return "LoggerPlugin";
});

accelerate.tap("Plugin2", (data) => {
  console.log(`上一个插件是: ${data}`);

  return "Plugin2";
});

accelerate.tap("Plugin3", (data) => {
  console.log(`上一个插件是: ${data}`);

  return "Plugin3";
});

const lastPlugin = accelerate.call(100);

console.log(`最后一个插件是：${lastPlugin}`);

```

## AsyncParallelHook
```
// 异步并发	不关心监听函数的返回值
const { AsyncParallelHook } = require("tapable");

const accelerate = new AsyncParallelHook(["newSpeed"]);

console.time("total time"); // 记录起始时间

// 注意注册异步事件需要使用tapPromise
// 回调函数要返回一个promise
accelerate.tapPromise("LoggerPlugin", (newSpeed) => {
  return new Promise((resolve) => {
    // 1秒后加速才完成
    setTimeout(() => {
      console.log("LoggerPlugin", `加速到${newSpeed}`);

      resolve();
    }, 1000);
  });
});

accelerate.tapPromise("OverspeedPlugin", (newSpeed) => {
  return new Promise((resolve) => {
    // 2秒后检测是否超速
    setTimeout(() => {
      if (newSpeed > 120) {
        console.log("OverspeedPlugin", "您已超速！！");
      }
      resolve();
    }, 2000);
  });
});

accelerate.tapPromise("DamagePlugin", (newSpeed) => {
  return new Promise((resolve) => {
    // 2秒后检测是否损坏
    setTimeout(() => {
      if (newSpeed > 300) {
        console.log("DamagePlugin", "速度实在太快，车子快散架了。。。");
      }

      resolve();
    }, 2000);
  });
});

// 触发事件使用promise，直接用then处理最后的结果
accelerate.promise(500).then(() => {
  console.log("任务全部完成");
  console.timeEnd("total time"); // 记录总共耗时
});
```

## AsyncParallelBailHook
```
// 异步并发	只要监听函数的返回值不为 null，就会忽略后面的监听函数执行，直接跳跃到callAsync等触发函数绑定的回调函数，然后执行这个被绑定的回调函数
const { AsyncParallelBailHook } = require("tapable");

const accelerate = new AsyncParallelBailHook(["newSpeed"]);

console.time("total time"); // 记录起始时间

accelerate.tapAsync("LoggerPlugin", (newSpeed, done) => {
  // 1秒后加速才完成
  setTimeout(() => {
    console.log("LoggerPlugin", `加速到${newSpeed}`);

    done();
  }, 1000);
});

accelerate.tapAsync("OverspeedPlugin", (newSpeed, done) => {
  // 2秒后检测是否超速
  setTimeout(() => {
    if (newSpeed > 120) {
      console.log("OverspeedPlugin", "您已超速！！");
    }

    // 这个任务的done返回一个错误
    // 注意第一个参数是node回调约定俗成的错误
    // 第二个参数才是Bail的返回值
    done(null, new Error("您已超速！！"));
  }, 2000);
});

accelerate.tapAsync("DamagePlugin", (newSpeed, done) => {
  // 3秒后检测是否损坏
  setTimeout(() => {
    if (newSpeed > 300) {
      console.log("DamagePlugin", "速度实在太快，车子快散架了。。。");
    }

    done();
  }, 3000);
});

accelerate.callAsync(500, (error, data) => {
  if (data) {
    console.log("任务执行出错：", data);
  } else {
    console.log("任务全部完成");
  }
  console.timeEnd("total time"); // 记录总共耗时
});


```

## AsyncSeriesHook
```
// 异步串行	不关心callback()的参数
const { AsyncSeriesHook } = require("tapable");

const accelerate = new AsyncSeriesHook(["newSpeed"]);

console.time("total time"); // 记录起始时间

accelerate.tapAsync("LoggerPlugin", (newSpeed, done) => {
  // 1秒后加速才完成
  setTimeout(() => {
    console.log("LoggerPlugin", `加速到${newSpeed}`);

    done();
  }, 1000);
});

accelerate.tapAsync("OverspeedPlugin", (newSpeed, done) => {
  // 2秒后检测是否超速
  setTimeout(() => {
    if (newSpeed > 120) {
      console.log("OverspeedPlugin", "您已超速！！");
    }
    done();
  }, 2000);
});

accelerate.tapAsync("DamagePlugin", (newSpeed, done) => {
  // 2秒后检测是否损坏
  setTimeout(() => {
    if (newSpeed > 300) {
      console.log("DamagePlugin", "速度实在太快，车子快散架了。。。");
    }

    done();
  }, 2000);
});

accelerate.callAsync(500, () => {
  console.log("任务全部完成");
  console.timeEnd("total time"); // 记录总共耗时
});


```

## AsyncSeriesBailHook
```
// 异步串行	callback()的参数不为null，就会直接执行callAsync等触发函数绑定的回调函数
const { AsyncSeriesBailHook } = require("tapable");

const accelerate = new AsyncSeriesBailHook(["newSpeed"]);

console.time("total time"); // 记录起始时间

accelerate.tapAsync("LoggerPlugin", (newSpeed, done) => {
  // 1秒后加速才完成
  setTimeout(() => {
    console.log("LoggerPlugin", `加速到${newSpeed}`);

    done();
  }, 1000);
});

accelerate.tapAsync("OverspeedPlugin", (newSpeed, done) => {
  // 2秒后检测是否超速
  setTimeout(() => {
    if (newSpeed > 120) {
      console.log("OverspeedPlugin", "您已超速！！");
    }

    // 这个任务的done返回一个错误
    // 注意第一个参数是node回调约定俗成的错误
    // 第二个参数才是Bail的返回值
    done(null, new Error("您已超速！！"));
  }, 2000);
});

accelerate.tapAsync("DamagePlugin", (newSpeed, done) => {
  // 2秒后检测是否损坏
  setTimeout(() => {
    if (newSpeed > 300) {
      console.log("DamagePlugin", "速度实在太快，车子快散架了。。。");
    }

    done();
  }, 2000);
});

accelerate.callAsync(500, (error, data) => {
  if (data) {
    console.log("任务执行出错：", data);
  } else {
    console.log("任务全部完成");
  }
  console.timeEnd("total time"); // 记录总共耗时
});


```

## AsyncSeriesWaterfallHook
```
// 异步串行	上一个监听函数的中的callback(err, data)的第二个参数,可以作为下一个监听函数的参数。
const { AsyncSeriesWaterfallHook } = require("tapable");

const accelerate = new AsyncSeriesWaterfallHook(["newSpeed"]);

console.time("total time"); // 记录起始时间

accelerate.tapAsync("LoggerPlugin", (newSpeed, done) => {
  // 1秒后加速才完成
  setTimeout(() => {
    console.log("LoggerPlugin", `加速到${newSpeed}`);

    // 注意done的第一个参数会被当做error
    // 第二个参数才是传递给后面任务的参数
    done(null, "LoggerPlugin");
  }, 1000);
});

accelerate.tapAsync("Plugin2", (data, done) => {
  setTimeout(() => {
    console.log(`上一个插件是: ${data}`);

    done(null, "Plugin2");
  }, 2000);
});

accelerate.tapAsync("Plugin3", (data, done) => {
  setTimeout(() => {
    console.log(`上一个插件是: ${data}`);

    done(null, "Plugin3");
  }, 2000);
});

accelerate.callAsync(500, (error, data) => {
  console.log("最后一个插件是:", data);
  console.timeEnd("total time"); // 记录总共耗时
});

```

## AsyncSeriesLoopHook
```
// 异步串行	可以触发handler循环调用。
```