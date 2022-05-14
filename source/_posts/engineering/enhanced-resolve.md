---
title: 【质量建设】Webpack - 路径解析库(enhanced-resolve)
categories: 工程化建设
tags: [Engineering]
comments: true
toc: true
---
## 路径解析
Webpack封装了一套解析库<font color=red>enhanced-resolve</font>专门用于解析路径，例如我们写了<font color=red>require('./index')</font>，Webpack在打包时就会用它来解析出./index的完整路径。
我们可以看到他的官方介绍：
```
Offers an async require.resolve function. It's highly configurable. Features:
- plug in system
- provide a custom filesystem
- sync and async node.js filesystems included
```

可以看到官方定义他是一个可配置化的异步require.resolve。如果不了解reqire.resolve的同学可以先看看[require.resolve](https://juejin.cn/post/6844904055806885895)是什么
## reqire.resolve的不足
可以看到本质上他们做的事情都是一样的，只是在解析路径的规则上enhanced-resolve提供了更强的扩展性，以满足Webpack对解析文件的需求。
由于require.resolve只能用在node的环境下，所以在设计时require.resolve只配置了node相关的依赖规则，而Webpack面对的环境多种多样，以下列举一些Webpack做的增强内容：
### 扩展名查询配置
例如解析./index时由于没有提供扩展名，所以它们都会去尝试遍历可能会有的文件，node会去尝试路径下是否有.js .json .node文件，而Webpack需要面对的文件扩展不止这三种，可通过配置扩展。
### 文件夹查询
require.resolve只会去解析文件的完整路径，但是enhanced-resolve既可以查询文件也可以查询文件夹。这个功能在Webpack中非常有用，可以通过它导入一个文件夹下的多个文件，只需要配置resolveToContext: true，就会尝试解析目录的完整路径。
### 返回结果
在解析成功时，require.resolve的返回值只有一个完整路径，enhanced-resolve的返回值还包含了描述文件等较为丰富的数据。
### 别名配置
node下使用别名是挺麻烦的事，但是enhanced-resolve非常好地支持了这个功能，即能让我们代码看上去更整洁，配合缓存还能提高解析效率。
## 使用方式
### 安装环境
可以到github: enhanced-resolve下载一份源码，本次我们使用4.1.1版本解析，下载完成后切换到对应分支，yarn add下载依赖模块。
在项目的package.json里，我们可以找到项目的导出文件是"main": "lib/node.js"，可以从这个文件入手开始探究源码。
### 使用
在主目录新建index.js，这里演示了常用的操作：
```
var resolve = require("./lib/node");
// 解析相对目录下index.js文件
resolve(__dirname, './index.js', (err, p, result) => {
    // p = /Users/enhanced-resolve/index.js
    console.log(err, p, result)
})
// 解析模块diff导出文件
resolve(__dirname, 'diff', (err, p, result) => {
    // p = /Users/enhanced-resolve/node_modules/diff/diff.js
    console.log(err, p, result)
})
// 解析绝对目录下的index.js文件
resolve(__dirname, '/Users/enhanced-resolve/index.js', (err, p, result) => {
    // p = /Users/enhanced-resolve/index.js
    console.log(err, p, result)
})
// 解析相对路径下的目录
resolve.context(__dirname, './', (err, p, result) => {
    // p = /Users/enhanced-resolve
    console.log(err, p, result)
})
```
### 打印调试信息
由于项目使用Tapable来组织流程，调试起来比较累，好在它提供的调试打印信息还算丰富，我们在Resolver.js第176行加上配置log: console.log，可以打印调试信息到控制台：
```
    return this.doResolve(
        this.hooks.resolve,
        obj,
        message,
        {
            missing: resolveContext.missing,
            stack: resolveContext.stack,
+           log: console.log,
        },
```
## 复制代码核心源码分析
### node.js
这个文件为我们提供了开箱即用的解析函数，根据不同场景预定义了默认参数，最终通过ResolverFactory.createResolver创建并执行路径解析，主要有以下三种场景：

- 文件路径解析：默认文件后缀为[".js", ".json", ".node"]
- 文件夹路径解析：只判断文件夹是否存在
- Loader路径解析：专门用于Webpack的Loader文件路径解析

另外每种场景都提供同步和异步调用方式，且默认文件操作通过CachedInputFileSystem包装提供缓存功能。
### ResolverFactory.js
主要做了两件事，一是参数解析，二是初始化插件，首先会根据参数来将需要用到的插件创建出来，调用他们的apply方法来初始化插件。
```
// ...
plugins = []
plugins.push(new ParsePlugin("resolve", "parsed-resolve"));
//...
plugins.push(new ResultPlugin(resolver.hooks.resolved));
plugins.forEach(plugin => {
    plugin.apply(resolver);
});
```
复制代码enhanced-resolve通过Tapable将所有插件串联起来，每个插件负责一件事情，通过事件流的方式传递每个插件的解析结果。所以只要看懂了插件之间的流转过程，就能明白它的工作原理。
### Resolver.js
这里是整个解析流程的核心，它继承了Tapable类，下面我们重点分析里面的方法。
```
class Resolver extends Tapable {
	constructor(fileSystem) {
		super();
		this.fileSystem = fileSystem;
		this.hooks = {
			resolve: new AsyncSeriesBailHook(["request", "resolveContext"])
		};
    }
}
```
#### ensureHook()
这个函数主要用与动态添加钩子，在构造函数中只定义了3个钩子，但是实际用到的不止这么少，所以在使用某个钩子前，都会调用这个方法保证钩子已经定义。这里创建的钩子都是AsyncSeriesBailHook类型，异步，串行执行，获取第一个返回值不为空的结果。
如果name的前缀带有before或after，则会调整调用优先级
```
ensureHook(name) {
    // ...
    const hook = this.hooks[name];
    if (!hook) {
        return this.hooks[name] =
            new AsyncSeriesBailHook(["request", "resolveContext"])
    }
    return hook;
}
```
例如有两个插件挂在一个钩子上，此时调用钩子hook.callAsync时，因为优先级高会先进入plugin-a的回调，如果返回值是空继续执行after-plugin-a的回调，否则直接执行hook.callAsync的回调：
```
this.ensureHook('myhook').tapAsync('after-plugin-a',() => {})
this.ensureHook('myhook').tapAsync('plugin-a',(request, resolveContext, callback) => {
    callback(null, 'ok from plugin-a')
})
this.hooks.myhook.callAsync(request, resolveContext, (err, result) => {
    console.log(result) // ok from plugin-a
})
```
#### doResolve()
这个函数是两个插件的链接点，核心很简单就是直接调用钩子执行下一个流程而已，源码中还有一大堆代码主要是用来记录日志。
hook.callAsync负责真正的调用，会开始执行挂在这个钩子上的事件。在钩子里又会调用doResolve执行下一个钩子。
```
doResolve(hook, request, message, resolveContext, callback) {
    // ...
    return hook.callAsync(request, innerContext, (err, result) => {
        if (err) return callback(err);
        if (result) return callback(null, result);
        callback();
    });
}
```
### XXXPlugin
在创建插件时一般会传入source和target两个参数：

- source：插件拿到Resolver.hooks['source']钩子，并调tap或tapAsync添加处理函数事件。当解析器接收到了source事件时，会执行注册的处理函数；
- target：在处理完毕后，调用doResolve触发一个target事件，交由下一个监听target事件的插件处理。
```
class ParsePlugin {
	constructor(source, target) {
		this.source = source;
		this.target = target;
	}
	apply(resolver) {
		const target = resolver.ensureHook(this.target);
		resolver.getHook(this.source)
			.tapAsync("ParsePlugin", (request, resolveContext, callback) => {
                // ...
				resolver.doResolve(target, obj, null, resolveContext, callback);
			});
	}
};
```
有了注册事件tapAsync和触发事件doResolve，各个插件就可以像积木一样链接起来
## 实战举例
我们通过分析一个解析目录是否存在的流程，来将上面内容串起来：
```
// index.js
resolve.context(__dirname, './', (err, p, result) => {
    // p = /Users/enhanced-resolve
    console.log(err, p, result)
})
```
打印出来的调试结果如下：
```
resolve './' in '/Users/enhanced-resolve'  
Parsed request is a directory  
using description file: /Users/enhanced-resolve/package.json (relative path: .)  
using description file: /Users/enhanced-resolve/package.json (relative path: .)  
as directory  
existing directory  
reporting result /Users/enhanced-resolve  
```
根据配置一共注册了以下插件，下面我们逐个分析这里用到的插件，插件流转路径如图示：
<img src="/images/engineering/webpack1.png" >
### ParsePlugin
用于预解析查询参数供后续插件使用：

解析查询路径中的query参数，如require('./index?id=1')，这是webpack特有的语法，见文档__resourceQuery;
判断查询路径是否是模块
判断查询路径是否是目录
```
parse(identifier) {
    const idxQuery = identifier.indexOf("?");
    const part = {
        request: identifier.slice(0, idxQuery),
        query: identifier.slice(idxQuery),
        file: false
    };
    part.module = this.isModule(part.request);
    part.directory = this.isDirectory(part.request);
    return part;
}
```
### DescriptionFilePlugin
用于获取描述文件路径，会在directory下搜索是否有package.json文件，如果该目录没有，就去上一级目录下查找，并计算出相对与directory的路径，下面是简化版的伪代码：
```
function loadDescriptionFile(directory, callback) {
    const descriptionFiles = ['package.json']
    let json
    let descriptionFilePath
    for(let i = 0; i < descriptionFiles.length; i++) {
        descriptionFilePath = path.join(directory, descriptionFiles[i])
        if(json = fileSystem.readJson(descriptionFilePath)) break
    }
    if(json === null) {
        directory = cdUp(directory)
        if (!directory) {
            return callback('err')
        } else {
            loadDescriptionFile(directory, callback);
            return
        }
    }
    const relativePath = "." + request.path.substr(result.directory.length).replace(/\\/g, "/");
    callback(null, {
        content: json,
        directory: directory,
        path: descriptionFilePath,
        relativePath,
    });
}
function cdUp(directory) {
	if (directory === "/") return null;
	const i = directory.lastIndexOf("/"),
		j = directory.lastIndexOf("\\");
	const p = i < 0 ? j : j < 0 ? i : i < j ? j : i;
	if (p < 0) return null;
	return directory.substr(0, p || 1);
}
```
### ModuleKindPlugin
如果ParsePlugin解析出来是模块路径，就引导至raw-module钩子，否则往下继续执行JoinRequestPlugin
```
(request, resolveContext, callback) => {
    if (!request.module) return callback();
    const obj = Object.assign({}, request);
    delete obj.module;
    resolver.doResolve(target, obj, "resolve as module", resolveContext, callback);
});
```
### JoinRequestPlugin
这里将会输出两个路径供后续查找：

path：指要查找的文件完整路径
relativePath：指描述文件路径相对于待查找文件的路径

关键输入参数：

request.path：在哪个路径下查找
request.request：查找的文件
request.relativePath：描述文件路径相对于request.path的路径
```
(request, resolveContext, callback) => {
    const obj = Object.assign({}, request, {
        path: resolver.join(request.path, request.request),
        relativePath: request.relativePath &&
            resolver.join(request.relativePath, request.request),
        request: undefined
    });
    resolver.doResolve(target, obj, null, resolveContext, callback);
});
```
### FileKindPlugin
如果ParsePlugin解析出来是路径，就往下继续执行TryNextPlugin，否则就引导至described-relative钩子
```
(request, resolveContext, callback) => {
    if (request.directory) return callback();
    const obj = Object.assign({}, request);
    delete obj.directory;
    resolver.doResolve(target, obj, null, resolveContext, callback)
```
### TryNextPlugin
直接引导到执行下一个插件
```
(request, resolveContext, callback) => {
    resolver.doResolve(target, obj, message, resolveContext, callback)
```
### DirectoryExistsPlugin
判断文件夹是否存在
```
(request, resolveContext, callback) => {
    const fs = resolver.fileSystem;
    const directory = request.path;
    fs.stat(directory, (err, stat) => {
        if (err || !stat || !stat.isDirectory()) {
            return callback();
        }
        resolver.doResolve(target, obj, "existing directory", resolveContext, callback)
    });
```
### NextPlugin
直接引导到执行下一个插件，和TryNextPlugin区别在于有没有携带message日志
```
(request, resolveContext, callback) => {
    resolver.doResolve(target, obj, null, resolveContext, callback)
```
### ResultPlugin
直接返回解析结果
```
(request, resolverContext, callback) => {
    // ...
    callback(null, request);
}
```
