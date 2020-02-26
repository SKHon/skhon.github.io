---
title: JavaScript Errors and Stack Traces
date: 2020-02-26 00:00:00
categories: 前端
tags: [node]
comments: true
copyright: true
---

## 介绍

今天将讨论错误和堆栈跟踪以及如何处理它们。

有时人们不会关注这些细节，但是，如果您正在编写与测试或错误相关的任何库，那么这些知识无疑会很有用。<!--more--> 例如，本周在 Chai，我们有一个很棒的 Pull Request，它极大地改善了我们处理堆栈跟踪的方式，以便用户在断言失败时获得更多信息。

通过操作堆栈跟踪，您可以清理无用的数据并集中精力处理重要的事情。 同样，在了解错误的确切含义及其属性后，您会更加自信地利用它。

这篇博客文章在开始时似乎太明显了，但是当我们开始操作堆栈跟踪时，它变得非常复杂，因此在移至该部分之前，请确保您对以前的内容有充分的了解。

## 调用堆栈如何工作

在讨论错误之前，我们必须了解调用堆栈的工作方式。 这确实很简单，但是在继续之前必须知道这一点。 如果您已经知道这一点，请随时跳过此部分。

每当有一个函数调用时，它就会被推到栈顶。 完成运行后，将其从堆栈顶部移除。

关于此数据结构的有趣之处在于，最后输入的将是第一个输出的数据。 这称为 LIFO（后进先出）属性。

这意味着，例如，当从函数 x 内部调用函数 y 时，我们将具有依次包含 x 和 y 的堆栈。

让我再举一个例子，假设您有以下代码：

```js
function c() {
  console.log("c");
}

function b() {
  console.log("b");
  c();
}

function a() {
  console.log("a");
  b();
}

a();
```

在上面的示例中，当运行 a 时，它将被添加到我们堆栈的顶部。 然后，当 b 从 a 内部被调用时，它被推入堆栈的顶部。 从 b 调用 c 时，也是如此。

运行 c 时，我们的堆栈跟踪将按此顺序包含 a，b 和 c。

一旦 c 完成运行，它就会从堆栈顶部移走，然后控制流回到 b。 当 b 完成时，它也将从堆栈中删除，现在我们将控件放回 a。 最后，当完成运行时，它也会从堆栈中删除。

为了更好地演示此行为，我们将使用 console.trace（），它将当前的堆栈跟踪信息打印到控制台。 同样，您通常应该从上至下读取堆栈跟踪。 将每行视为从其下一行内部调用的内容。

```js
function c() {
  console.log("c");
  console.trace();
}

function b() {
  console.log("b");
  c();
}

function a() {
  console.log("a");
  b();
}

a();
```

在 Node REPL 服务器中运行此命令时，我们会得到以下信息：

```js
Trace
    at c (repl:3:9)
    at b (repl:3:1)
    at a (repl:3:1)
    at repl:1:1 // <-- For now feel free to ignore anything below this point, these are Node's internals
    at realRunInThisContextScript (vm.js:22:35)
    at sigintHandlersWrap (vm.js:98:12)
    at ContextifyScript.Script.runInThisContext (vm.js:24:12)
    at REPLServer.defaultEval (repl.js:313:29)
    at bound (domain.js:280:14)
    at REPLServer.runBound [as eval] (domain.js:293:12)
```

正如我们在这里看到的，当从 c 内部打印堆栈时，我们有 a，b 和 c。

现在，如果在 c 完成运行之后从 b 内部打印堆栈跟踪，我们将能够看到它已从堆栈顶部删除，因此我们只有 a 和 b。

```js
function c() {
  console.log("c");
}

function b() {
  console.log("b");
  c();
  console.trace();
}

function a() {
  console.log("a");
  b();
}

a();
```

如您所见，我们的堆栈中不再有 c，因为它已经完成运行并已弹出。

```js
Trace
    at b (repl:4:9)
    at a (repl:3:1)
    at repl:1:1  // <-- For now feel free to ignore anything below this point, these are Node's internals
    at realRunInThisContextScript (vm.js:22:35)
    at sigintHandlersWrap (vm.js:98:12)
    at ContextifyScript.Script.runInThisContext (vm.js:24:12)
    at REPLServer.defaultEval (repl.js:313:29)
    at bound (domain.js:280:14)
    at REPLServer.runBound [as eval] (domain.js:293:12)
    at REPLServer.onLine (repl.js:513:10)
```

简而言之：您调用事物，它们就会被推到栈顶。 当他们完成运行时，它们会弹出。 就那么简单。

## 错误对象和错误处理

当发生错误时，通常会抛出一个 Error 对象。 错误对象也可以用作想要扩展它并创建自己的错误的用户的原型。

Error.prototype 对象通常具有以下属性：

- constructor - The constructor function responsible for this instance’s prototype.
- message - An error message.
- name - The error’s name.

这些是标准属性，有时每个环境都有其自己的特定属性。 在某些环境中，例如 Node，Firefox，Chrome，Edge，IE 10 +，Opera 和 Safari 6+，我们甚至拥有 stack 属性，其中包含错误的堆栈跟踪。 错误的堆栈跟踪包含所有堆栈帧，直到其自己的构造函数为止。

如果您想阅读有关 Error 对象的特定属性的更多信息，我强烈建议您阅读有关 MDN 的文章。

要抛出错误，必须使用 throw 关键字。 为了捕获引发的错误，必须将可能引发错误的代码包装到 try 块中，然后再放入 catch 块。 Catch 还接受一个参数，即抛出的错误。

正如 Java 中所发生的那样，JavaScript 还允许您在 try / catch 块之后运行一个 finally 块，而不管您的 try 块是否抛出错误。 处理完毕后，最好使用最后清理的东西，无论您的操作是否有效。

到目前为止，对于大多数人来说，一切都是显而易见的，所以让我们来谈谈一些不重要的细节。

您可以使用 try 块，但不要在 catch 后面，但是必须在它们后面紧跟着。 这意味着我们可以使用三种不同形式的 try 语句：

- try...catch
- try...finally
- try...catch...finally

try 语句可以嵌套在其他 try 语句中，例如：

```js
try {
  try {
    throw new Error("Nested error."); // The error thrown here will be caught by its own `catch` clause
  } catch (nestedErr) {
    console.log("Nested catch"); // This runs
  }
} catch (err) {
  console.log("This will not run.");
}
```

您还可以将 try 语句嵌套到 catch 和 finally 块中：

```js
try {
  throw new Error("First error");
} catch (err) {
  console.log("First catch running");
  try {
    throw new Error("Second error");
  } catch (nestedErr) {
    console.log("Second catch running.");
  }
}
try {
  console.log("The try block is running...");
} finally {
  try {
    throw new Error("Error inside finally.");
  } catch (err) {
    console.log("Caught an error inside the finally block.");
  }
}
```

请务必注意，您还可以抛出不是 Error 对象的值，这一点也很重要。 尽管这看起来很酷，但实际上并没有那么好，尤其是对于那些使用必须处理其他人的代码的库的开发人员而言，因为那时没有标准，而且您永远都不知道用户会期望什么。 您不能仅仅因为它们选择不这样做而仅仅抛出一个字符串或一个数字就相信它们会引发 Error 对象。 如果您需要处理堆栈跟踪和其他有意义的元数据，这也将变得更加困难。

假设您有以下代码：

```js
function runWithoutThrowing(func) {
  try {
    func();
  } catch (e) {
    console.log("There was an error, but I will not throw it.");
    console.log("The error's message was: " + e.message);
  }
}

function funcThatThrowsError() {
  throw new TypeError("I am a TypeError.");
}

runWithoutThrowing(funcThatThrowsError);
```

如果您的用户正在传递将 Error 对象抛出到您的 runWithoutThrowing 函数的函数，这将非常有用。 但是，如果它们最终抛出一个 String，那么您可能会遇到麻烦：

```js
function runWithoutThrowing(func) {
  try {
    func();
  } catch (e) {
    console.log("There was an error, but I will not throw it.");
    console.log("The error's message was: " + e.message);
  }
}

function funcThatThrowsString() {
  throw "I am a String.";
}

runWithoutThrowing(funcThatThrowsString);
```

现在，您的第二个 console.log 将显示错误消息未定义。 现在这似乎并不重要，但是如果您需要确保某个 Error 对象上存在某些属性或以另一种方式处理特定于 Error 的属性（例如 Chai 的 throws 断言确实需要），则您需要做更多的工作来确保它可以工作 对。

另外，当抛出不是 Error 对象的值时，您将无权访问其他重要数据，例如堆栈，这是 Error 对象在某些环境中具有的属性。

错误也可以用作任何其他对象，您不一定需要将其引发，这就是为什么它们多次被用作回调函数的第一个参数的原因，例如 fs.readdir 函数就是这种情况。

```js
const fs = require("fs");

fs.readdir("/example/i-do-not-exist", function callback(err, dirs) {
  if (err instanceof Error) {
    // `readdir` will throw an error because that directory does not exist
    // We will now be able to use the error object passed by it in our callback function
    console.log("Error Message: " + err.message);
    console.log("See? We can use Errors without using try statements.");
  } else {
    console.log(dirs);
  }
});
```

最后但并非最不重要的一点是，在拒绝承诺时也可以使用 Error 对象。 这使得处理承诺拒绝更容易：

```js
new Promise(function(resolve, reject) {
  reject(new Error("The promise was rejected."));
})
  .then(function() {
    console.log("I am an error.");
  })
  .catch(function(err) {
    if (err instanceof Error) {
      console.log("The promise was rejected with an error.");
      console.log("Error Message: " + err.message);
    }
  });
```

## 操纵堆栈跟踪

现在，您一直在等待的部分：如何操作堆栈跟踪。

本章专门针对支持 Error.captureStackTrace 的环境（例如 NodeJS）。

Error.captureStackTrace 函数将一个对象作为第一个参数，并可选地将一个函数作为第二个参数。 捕获堆栈跟踪的作用是（显然）捕获当前堆栈跟踪，并在目标对象中创建一个堆栈属性来存储它。 如果提供了第二个参数，则传递的函数将被视为调用堆栈的终点，因此，堆栈跟踪将仅显示在调用此函数之前发生的调用。

让我们使用一些示例来更清楚地说明这一点。 首先，我们将捕获当前的堆栈跟踪并将其存储在一个公共对象中。

```js
const myObj = {};

function c() {}

function b() {
  // Here we will store the current stack trace into myObj
  Error.captureStackTrace(myObj);
  c();
}

function a() {
  b();
}

// First we will call these functions
a();

// Now let's see what is the stack trace stored into myObj.stack
console.log(myObj.stack);

// This will print the following stack to the console:
//    at b (repl:3:7) <-- Since it was called inside B, the B call is the last entry in the stack
//    at a (repl:2:1)
//    at repl:1:1 <-- Node internals below this line
//    at realRunInThisContextScript (vm.js:22:35)
//    at sigintHandlersWrap (vm.js:98:12)
//    at ContextifyScript.Script.runInThisContext (vm.js:24:12)
//    at REPLServer.defaultEval (repl.js:313:29)
//    at bound (domain.js:280:14)
//    at REPLServer.runBound [as eval] (domain.js:293:12)
//    at REPLServer.onLine (repl.js:513:10)
```

您可以在上面的示例中注意到，我们首先调用 a（将其推入堆栈），然后从 a 内部调用 b（将其推入 a 顶部）。 然后，在 b 内部，我们捕获了当前的堆栈跟踪并将其存储到 myObj 中。 这就是为什么我们在打印到控制台的堆栈中只得到 a 然后再得到 b 的原因。

现在，将一个函数作为第二个参数传递给 Error.captureStackTrace 函数，看看会发生什么：

```js
const myObj = {};

function d() {
  // Here we will store the current stack trace into myObj
  // This time we will hide all the frames after `b` and `b` itself
  Error.captureStackTrace(myObj, b);
}

function c() {
  d();
}

function b() {
  c();
}

function a() {
  b();
}

// First we will call these functions
a();

// Now let's see what is the stack trace stored into myObj.stack
console.log(myObj.stack);

// This will print the following stack to the console:
//    at a (repl:2:1) <-- As you can see here we only get frames before `b` was called
//    at repl:1:1 <-- Node internals below this line
//    at realRunInThisContextScript (vm.js:22:35)
//    at sigintHandlersWrap (vm.js:98:12)
//    at ContextifyScript.Script.runInThisContext (vm.js:24:12)
//    at REPLServer.defaultEval (repl.js:313:29)
//    at bound (domain.js:280:14)
//    at REPLServer.runBound [as eval] (domain.js:293:12)
//    at REPLServer.onLine (repl.js:513:10)
//    at emitOne (events.js:101:20)
```

当我们将 b 传递给 Error.captureStackTraceFunction 时，它会将 b 自身及其上方的所有帧都隐藏起来。 这就是为什么我们在堆栈跟踪中仅包含一个的原因。

现在您可能会问自己：“为什么这样做有用？”。 这很有用，因为您可以使用它来隐藏与用户无关的内部实现细节。 例如，在 Chai 中，我们使用它来避免向用户显示与我们自己实现检查和断言的方式无关的细节。

## 现实世界中的堆栈跟踪操作

正如我在上一节中提到的，Chai 使用堆栈操纵技术使堆栈跟踪与我们的用户更加相关。 这是我们的方法。

首先，让我们看一下断言失败时抛出的 AssertionError 构造函数：

```js
// `ssfi` stands for "start stack function". It is the reference to the
// starting point for removing irrelevant frames from the stack trace
function AssertionError(message, _props, ssf) {
  var extend = exclude("name", "message", "stack", "constructor", "toJSON"),
    props = extend(_props || {});

  // Default values
  this.message = message || "Unspecified AssertionError";
  this.showDiff = false;

  // Copy from properties
  for (var key in props) {
    this[key] = props[key];
  }

  // Here is what is relevant for us:
  // If a start stack function was provided we capture the current stack trace and pass
  // it to the `captureStackTrace` function so we can remove frames that come after it
  ssf = ssf || arguments.callee;
  if (ssf && Error.captureStackTrace) {
    Error.captureStackTrace(this, ssf);
  } else {
    // If no start stack function was provided we just use the original stack property
    try {
      throw new Error();
    } catch (e) {
      this.stack = e.stack;
    }
  }
}
```

如上所示，我们使用 Error.captureStackTrace 捕获堆栈跟踪并将其存储到我们正在构建的 AssertionError 实例中（如果存在），我们将向其传递一个起始堆栈函数，以从中删除无关帧 堆栈跟踪，仅显示 Chai 的内部实现细节，最终使堆栈“变脏”。

现在，让我们看一下@meeber 在此出色的 PR 中编写的最新代码。

在查看下面的代码之前，我必须告诉您 addChainableMethod 的作用。 它将传递给它的可链接方法添加到断言中，并且还使用包装断言的方法标记断言本身。 它以名称 ssfi（代表启动堆栈功能指示器）存储。 这基本上意味着当前断言将是堆栈中的最后一帧，因此我们将在堆栈中不再显示 Chai 的其他内部方法。 我避免为此添加完整的代码，因为它可以完成很多事情并且有点棘手，但是如果您想阅读它，请转到此处的链接。

在下面的代码段中，我们具有 lengthOf 断言的逻辑，该逻辑检查对象是否具有特定长度。 我们希望我们的用户像这样使用它：expect(['foo'，'bar']).to.have.lengthOf(2)。

```js
function assertLength(n, msg) {
  if (msg) flag(this, "message", msg);
  var obj = flag(this, "object"),
    ssfi = flag(this, "ssfi");

  // Pay close attention to this line
  new Assertion(obj, msg, ssfi, true).to.have.property("length");
  var len = obj.length;

  // This line is also relevant
  this.assert(
    len == n,
    "expected #{this} to have a length of #{exp} but got #{act}",
    "expected #{this} to not have a length of #{act}",
    n,
    len
  );
}

Assertion.addChainableMethod("lengthOf", assertLength, assertLengthChain);
```

在上面的代码中，我突出显示了当前与我们相关的行。 让我们从对 this.assert 的调用开始。

这是 this.assert 方法的代码：

```js
Assertion.prototype.assert = function(
  expr,
  msg,
  negateMsg,
  expected,
  _actual,
  showDiff
) {
  var ok = util.test(this, arguments);
  if (false !== showDiff) showDiff = true;
  if (undefined === expected && undefined === _actual) showDiff = false;
  if (true !== config.showDiff) showDiff = false;

  if (!ok) {
    msg = util.getMessage(this, arguments);
    var actual = util.getActual(this, arguments);

    // This is the relevant line for us
    throw new AssertionError(
      msg,
      {
        actual: actual,
        expected: expected,
        showDiff: showDiff
      },
      config.includeStack ? this.assert : flag(this, "ssfi")
    );
  }
};
```

基本上，assert 方法负责检查 assert 布尔表达式是否通过。 如果没有，我们必须实例化一个 AssertionError。 请注意，在实例化此新的 AssertionError 时，我们还向其传递了堆栈跟踪功能指示器（ssfi）。 如果打开了配置标记 includeStack，我们通过将 this.assert 本身传递给用户，向用户显示整个堆栈跟踪，这实际上是堆栈中的最后一帧。 但是，如果关闭了 includeStack 配置标志，我们必须从堆栈跟踪中隐藏更多内部实现细节，因此我们将使用存储在 ssfi 标志中的内容。

现在，让我们谈谈与我们相关的另一条线：

```js
new Assertion(obj, msg, ssfi, true).to.have.property("length");
```

如您所见，在创建嵌套断言时，我们将传递从 ssfi 标志获得的内容。 这意味着在创建新的断言时，它将使用此函数作为从堆栈跟踪中删除无用帧的起点。 顺便说一下，这是 Assertion 构造函数：

```js
function Assertion(obj, msg, ssfi, lockSsfi) {
  // This is the line that matters to us
  flag(this, "ssfi", ssfi || Assertion);
  flag(this, "lockSsfi", lockSsfi);
  flag(this, "object", obj);
  flag(this, "message", msg);

  return util.proxify(this);
}
```

您可以从我对 addChainableMethod 的描述中记住，它使用自己的 wrapper 方法设置 ssfi 标志，这意味着这是堆栈跟踪中的最低内部框架，因此我们可以删除其上方的所有框架。

通过将 ssfi 传递给嵌套断言，该断言仅检查我们的对象是否具有属性长度，从而避免了重置框架（该框架将用作起点指示符），而使先前的 addChainableMethod 在堆栈中可见。

这看起来似乎有些复杂，所以让我们回顾一下 Chai 内部发生的情况，我们想从堆栈中删除无用的帧：

- 当运行断言时，我们将其自己的方法设置为删除堆栈中下一帧的参考
- 断言运行，如果失败，我们将在存储引用后删除所有内部框架
- 如果我们嵌套了断言，则仍必须使用当前断言包装器方法作为删除堆栈中下一帧的参考点，因此我们将当前的 ssfi（启动栈功能指示器）传递给我们正在创建的断言，以便可以保留它

我也强烈建议您阅读@meeber 的评论，以了解它。
