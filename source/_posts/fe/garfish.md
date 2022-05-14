---
title: 【源码解析】探索garfish源码
categories: 前端
tags: [Web]
comments: true
copyright: true
toc: true
---
# 准备工作
首先，可以在github上把代码拉到本地，地址为：https://github.com/modern-js-dev/garfish
然后就需要把项目跑起来，方便调试。总共四步即可：
1. 全局安装pnpm：
> $ npm i -g pnpm
2. 安装依赖
> $ pnpm install
3. 启动build:watch
> $ pnpm build:watch
4. 启动dev
> $ pnpm dev
这样就跑起来了。整体框架代码讲解主要以流程为主，一些细节性逻辑就直接跳过，有兴趣的同学可以翻源码。
# 在主应用中引入Garfish实例：
在主应用中，首先引入Garfish实例：
```
// dev/main/src/index.ts
import GarfishInstance from 'garfish';
```
先看一下这个实例是个啥，追溯一下源码，发现Garfish实例是一个函数返回的：
```
// packages/garfish/src/index.ts
function createContext(): Garfish {
  // ...
  // Existing garfish instance, direct return
  if (inBrowser() && window['__GARFISH__'] && window['Garfish']) {
    return window['Garfish'];
  }

  const GarfishInstance = new Garfish({
    plugins: [GarfishRouter(), GarfishBrowserVm(), GarfishBrowserSnapshot()],
  });

  // ...
  
  return GarfishInstance;
}
```
createContext可以理解为创建上下文环境，从上至下捋一下，如果是浏览器环境，并且window上存在Garfish对象，则直接返回（熟悉设计模式的同学，应该知道这是一个单例模式）。接下来new了一个Garfish对象，并且默认传入了一个对象 { plugins: [GarfishRouter(), GarfishBrowserVm(), GarfishBrowserSnapshot()] }。这是初始化了三个插件，我们继续看一下这三个插件长什么样子。
我们先看一下GarfishRouter
```
// packages/router/src/index.ts
export function GarfishRouter(_args?: Options) {
  return function (Garfish: interfaces.Garfish): interfaces.Plugin {
    Garfish.apps = {};
    Garfish.router = router;

    return {
      name: 'router',
      version: __VERSION__,

      bootstrap(options: interfaces.Options) {
        // ...
      },

      registerApp(appInfos) {
        // ...
      },
    };
  };
}
```
可以看到GarfishRouter()返回的是一个function，形成闭包，最后会返回一个对象。这个对象其实是一个插件的格式，有name、version，还有生命周期钩子bootstrap、registerApp，生命周期钩子我们后续会介绍到，这里大家了解就行。再看看其他两个插件是什么：
GarfishBorwserVm:
```
// packages/browser-vm/src/pluginify.ts
export function GarfishBrowserVm() {
  return function (Garfish: interfaces.Garfish): interfaces.Plugin {
    Garfish.getGlobalObject = function () {
      return Sandbox.getNativeWindow();
    };

    Garfish.setGlobalValue = function (key, value) {
      return (this.getGlobalObject()[key] = value);
    };

    Garfish.clearEscapeEffect = function (key, value) {
      const global = this.getGlobalObject();
      if (key in global) {
        global[key] = value;
      }
    };
    return createOptions(Garfish);
  };
}

function createOptions(Garfish: interfaces.Garfish) {
  // ...
  const options: interfaces.Plugin = {
    name: 'browser-vm',
    version: __VERSION__,

    afterLoad(appInfo, appInstance) {
      // ...
    },

    // If the app is uninstalled, the sandbox needs to clear all effects and then reset
    afterUnmount(appInfo, appInstance, isCacheMode) {
      // ...
    },

    afterMount(appInfo, appInstance) {
      // ...
    },
  };
  return options;
}
```
GarfishBrowserSnapshot: 
```
// packages/browser-snapshot/src/index.ts
export function GarfishBrowserSnapshot(op?: BrowserConfig) {
  return function (Garfish: interfaces.Garfish): interfaces.Plugin {
    const config: BrowserConfig = op || { open: true };

    const options = {
      openBrowser: false,
      version: __VERSION__,
      name: 'browser-snapshot',

      afterLoad(appInfo, appInstance) {
        // ...
      },

      beforeMount(appInfo, appInstance) {
        // ...
      },

      afterUnmount(appInfo, appInstance) {
        // ...
      },
    };
    return options;
  };
}
```
可以看到都是返回一个函数，并且这个函数的返回格式有点类似，其实这就是garfish插件的形式，返回一个函数，该函数返回一个对象，这个对象包含了这个插件的一些信息，可以总结成这样：
```
{
    name: '',
    version: '',
    lifecycle: '' // 这里的lifecycle是泛指生命周期的钩子函数，具体指bootstrap、beforeBootstrap等等
    ...
}
```
插件中生命周期的具体实现，放在后面讲述。这里暂时把代码运行链路拉通。接下来再返回Garfish类的实现，它的实例是什么样子的。
Garfish类的实现：
```
// packages/core/src/garfish.ts
export class Garfish extends EventEmitter2 {
  public running = false;
  public version = __VERSION__;
  public flag = __GARFISH_FLAG__; // A unique identifier
  public loader = new Loader();
  public hooks = globalLifecycle();
  public channel = new EventEmitter2();
  public options = createDefaultOptions();
  public externals: Record<string, any> = {};
  public activeApps: Array<interfaces.App> = [];
  public plugins: interfaces.Plugins = {} as any;
  public cacheApps: Record<string, interfaces.App> = {};
  public appInfos: Record<string, interfaces.AppInfo> = {};

  private nestedSwitch = false;
  private loading: Record<string, Promise<any> | null> = {};

  get props(): Record<string, any> {
    return (this.options && this.options.props) || DEFAULT_PROPS.get(this);
  }

  constructor(options: interfaces.Options) {
    super();
    this.setOptions(options);
    DEFAULT_PROPS.set(this, {});
    this.options.plugins?.forEach((plugin) => this.usePlugin(plugin));
  }

  private setOptions(options: Partial<interfaces.Options>) {
    // ...
  }

  createPluginSystem<T extends (api: typeof HOOKS_API) => any>(callback: T) {
    // ...
  }

  usePlugin(
    plugin: (context: Garfish) => interfaces.Plugin,
    ...args: Array<any>
  ) {
    // ...
  }

  run(options: interfaces.Options = {}) {
    // ...
  }

  registerApp(list: interfaces.AppInfo | Array<interfaces.AppInfo>) {
    // ...
  }

  setExternal(nameOrExtObj: string | Record<string, any>, value?: any) {
    // ...
  }

  async loadApp(
    appName: string,
    optionsOrUrl?: Omit<interfaces.AppInfo, 'name'>,
  ): Promise<interfaces.App | null> {
    // ...
  }
}
```
先看一下钩子函数中，做了什么：
```
// packages/core/src/garfish.ts
constructor(options: interfaces.Options) {
    super();
    // 传入的options（其实就是默认的那三个插件） 深度merge到默认的options上。
    this.setOptions(options);
    DEFAULT_PROPS.set(this, {});
    // 这里分别执行默认的三个插件
    this.options.plugins?.forEach((plugin) => this.usePlugin(plugin));
  }
```
this.setOptions(options)主要是把传入的options深度merge到默认的options中。而传入的options就是传入的那三个默认插件。我们看一下默认的options有哪些属性吧：
```
// packages/core/src/config.ts
export const createDefaultOptions = (nested = false) => {
  const config: interfaces.Options = {
    // global config
    appID: '',
    apps: [],
    autoRefreshApp: true,
    disableStatistics: false,
    disablePreloadApp: false,
    // app config
    basename: '/',
    props: {},
    // Use an empty div by default
    domGetter: () => document.createElement('div'),
    sandbox: {
      snapshot: false,
      disableWith: false,
      strictIsolation: false,
    },
    // global hooks
    beforeLoad: () => {},
    afterLoad: () => {},
    errorLoadApp: (e) => error(e),
    // Router
    onNotMatchRouter: () => {},
    // app hooks
    // Code eval hooks
    beforeEval: () => {},
    afterEval: () => {},
    // App mount hooks
    beforeMount: () => {},
    afterMount: () => {},
    beforeUnmount: () => {},
    afterUnmount: () => {},
    // Error hooks
    errorMountApp: (e) => error(e),
    errorUnmountApp: (e) => error(e),
    customLoader: null, // deprecated
  };

  if (nested) {
    invalidNestedAttrs.forEach((key) => delete config[key]);
  }
  return config;
};
```
再回到构造函数中，接下来分别对plugins进行usePlugin操作，其实就是为了拿到插件中的一些属性，并且进行一些注册操作。我们看一下usePlugin做了些什么：
```
// packages/core/src/garfish.ts
usePlugin(
    plugin: (context: Garfish) => interfaces.Plugin,
    ...args: Array<any>
  ) {
   // ...
   
    // this指向是Garfish类
    args.unshift(this); 
    
    // 执行传入的plugin
    const pluginConfig = plugin.apply(null, args) as interfaces.Plugin;
    assert(pluginConfig.name, 'The plugin must have a name.');

    // 如果没有注册过，则进行注册 
    if (!this.plugins[pluginConfig.name]) {
      this.plugins[pluginConfig.name] = pluginConfig;
      // Register hooks, Compatible with the old api
      this.hooks.usePlugin(pluginConfig);
    } else if (__DEV__) {
      warn('Please do not register the plugin repeatedly.');
    }
    return this;
  }
```
args数组首位是Garfish自己，然后获取插件配置，前面我们提到了，插件最后的返回是一个对象，里面包含name、version、生命周期钩子等。然后plugin.apply()就是返回的这些配置，并且赋值给了pluginConfig。接下来就是注册逻辑了，如果之前没有注册过该plugin，则进行注册，就是key为plugin的name，value为具体的配置形式，放在this.plugins对象中，这个好理解，接下来是进行this.hooks.usePlugin(pluginConfig)操作，这个其实是用来注册生命周期的，看一下this.hooks是啥，再构造函数中，是这么初始化hooks的：
// packages/core/src/garfish.ts
public hooks = globalLifecycle();
通过函数名，也应该能猜得到，全局生命周期的hooks。接着看globalLifeCycle实现：
```
// packages/core/src/lifecycle.ts
export function globalLifecycle() {
  return new PluginSystem({
    beforeBootstrap: new SyncHook<[interfaces.Options], void>(),
    bootstrap: new SyncHook<[interfaces.Options], void>(),
    beforeRegisterApp: new SyncHook<[interfaces.AppInfo | Array<interfaces.AppInfo>], void>(),
    registerApp: new SyncHook<[Record<string, interfaces.AppInfo>], void>(),
    beforeLoad: new AsyncHook<[interfaces.AppInfo], Promise<boolean | void> | void | boolean>(),
    afterLoad: new AsyncHook<[interfaces.AppInfo, interfaces.App], void>(),
    errorLoadApp: new SyncHook<[Error, interfaces.AppInfo], void>(),
  });
}
```
返回的是一个pluginSystem实例，传入一个对象，包含7个生命周期属性。先看pluginSystem类的实现：
```
// packages/hooks/src/pluginSystem.ts
export class PluginSystem<T extends Record<string, any>> {
  lifecycle: T;
  lifecycleKeys: Array<keyof T>;
  private registerPlugins: Record<string, Plugin<T>> = {};

  constructor(lifecycle: T) {
    /*
      lifecycle: 
      {
        beforeBootstrap: new SyncHook<[interfaces.Options], void>(),
        bootstrap: new SyncHook<[interfaces.Options], void>(),
        beforeRegisterApp: new SyncHook<[interfaces.AppInfo | Array<interfaces.AppInfo>], void>(),
        registerApp: new SyncHook<[Record<string, interfaces.AppInfo>], void>(),
        beforeLoad: new AsyncHook<[interfaces.AppInfo], Promise<boolean | void> | void | boolean>(),
        afterLoad: new AsyncHook<[interfaces.AppInfo, interfaces.App], void>(),
        errorLoadApp: new SyncHook<[Error, interfaces.AppInfo], void>(),
      }
    */
    this.lifecycle = lifecycle;
    this.lifecycleKeys = Object.keys(lifecycle);
  }

  usePlugin(plugin: Plugin<T>) {
    assert(isPlainObject(plugin), 'Invalid plugin configuration.');
    // Plugin name is required and unique
    const pluginName = plugin.name;
    assert(pluginName, 'Plugin must provide a name.');

    if (!this.registerPlugins[pluginName]) {
      this.registerPlugins[pluginName] = plugin;

      for (const key in this.lifecycle) {
        const pluginLife = plugin[key as string];
        if (pluginLife) {
          // Differentiate different types of hooks and adopt different registration strategies
          this.lifecycle[key].on(pluginLife);
        }
      }
    } else if (__DEV__) {
      warn(`Repeat to register plugin hooks "${pluginName}".`);
    }
  }

  removePlugin(pluginName: string) {
    assert(pluginName, 'Must provide a name.');
    const plugin = this.registerPlugins[pluginName];
    assert(plugin, `plugin "${pluginName}" is not registered.`);

    for (const key in plugin) {
      this.lifecycle[key].remove(plugin[key as string]);
    }
  }
}
```
可以看到上面提到的this.hooks.usePlugin其实是执行了pluginSystem类中的usePlugin方法，实现逻辑也是会在pluginSystem类中的registerPlugins进行注册，然后会在全局生命周期的不同钩子上，注册每个插件配置中对应的钩子函数。这个大家需要好好理解一下，这个设计我们在写自己的框架时可以学习一下。再返回到创建pluginSystem实例中，传入的几个hooks，主要包含两种，一种是同步的SyncHook，另一种是异步的AsyncHook。看一下这两类hooks的实现。
SyncHook：
```
// packages/hooks/src/syncHook.ts
export class SyncHook<T, K> {
  public type: string = '';
  public listeners = new Set<Callback<T, K>>();

  constructor(type?: string) {
    if (type) this.type = type;
  }

  on(fn: Callback<T, K>) {
    if (typeof fn === 'function') {
      this.listeners.add(fn);
    } else if (__DEV__) {
      warn('Invalid parameter in "Hook".');
    }
  }

  once(fn: Callback<T, K>) {
    const self = this;
    this.on(function wrapper(...args: Array<any>) {
      self.remove(wrapper);
      return fn.apply(null, args);
    });
  }

  emit(...data: ArgsType<T>) {
    if (this.listeners.size > 0) {
      this.listeners.forEach((fn) => fn.apply(null, data));
    }
  }

  remove(fn: Callback<T, K>) {
    return this.listeners.delete(fn);
  }

  removeAll() {
    this.listeners.clear();
  }
}
```
就是一个简单的发布订阅模式。
AsyncHook：
```
// packages/hooks/src/asyncHook.ts
export class AsyncHook<T, K> extends SyncHook<T, K> {
  emit(...data: ArgsType<T>): Promise<void | false> {
    let result;
    const ls = Array.from(this.listeners);
    if (ls.length > 0) {
      let i = 0;
      const call = (prev?: any) => {
        if (prev === false) {
          return false; // Abort process
        } else if (i < ls.length) {
          return Promise.resolve(ls[i++].apply(null, data)).then(call);
        }
      };
      result = call();
    }
    return Promise.resolve(result);
  }
}
```
AsyncHook的emit实现，可以理解将一串的异步函数，进行同步处理，上一个异步函数的返回，是下一个异步函数的入参，如果看过Koa中间件的实现，就很容易明白这样逻辑的实现了。
到目前为止，当主应用引入Garfish后，初步的注册工作基本完成了。
# 主应用中启动Garfish
我们先假设主应用默认路由就是/，不默认展示子应用。
再看主应用如何进行下一步的：
```
// main/src/index.ts
import GarfishInstance from 'garfish'; 
import { Config } from './config';

GarfishInstance.run(Config);
第一行代码，我们已经在上面介绍过了，继续往下走，看看Config是什么：
// dev/main/src/config.ts
let defaultConfig: interfaces.Options = {
  basename: '/garfish_master',
  domGetter: () => {
    // await asyncTime();
    return document.querySelector('#submoduleByRouter');
  },
  apps: [
    // {
    //   name: 'vue',
    //   activeWhen: '/vue',
    //   cache: false,
    //   entry: 'http://localhost:2666',
    // },
    {
      name: 'vue2',
      cache: false,
      activeWhen: '/vue2',
      entry: 'http://localhost:2777',
    },
    {
      name: 'react',
      activeWhen: '/react',
      entry: 'http://localhost:2444',
      props: {
        appName: 'react',
      },
    },
  ],
  autoRefreshApp: false,
  disablePreloadApp: true,
  protectVariable: ['MonitoringInstance', 'Garfish'],
  sandbox: {
    open: true,
    // strictIsolation: true,
  },

  // beforeMount(appInfo) {
  //   console.log('beforeMount', appInfo);
  // },

  // afterLoad(info, app) {
  //   console.log(app.vmSandbox);
  // },

  customLoader() {},
};
```
这个默认配置，是用户可以自定义的，通过之前的源码分析，这些配置最后会深度merge到Garfish类中的默认配置。接下来就是GarfishInstance.run(Config)，将配置传入，然后执行Garfish类中的run方法。
```
// packages/core/src/garfish.ts
run(options: interfaces.Options = {}) {
    if (this.running) {
      // ...
    }

    this.setOptions(options);
    // Register plugins
    this.usePlugin(GarfishHMRPlugin());
    this.usePlugin(GarfishPerformance());
    if (!this.options.disablePreloadApp) {
      this.usePlugin(GarfishPreloadPlugin());
    }
    options.plugins?.forEach((plugin) => this.usePlugin(plugin));
    // Put the lifecycle plugin at the end, so that you can get the changes of other plugins
    this.usePlugin(GarfishOptionsLife(this.options, 'global-lifecycle'));

    // Emit hooks and register apps
    this.hooks.lifecycle.beforeBootstrap.emit(this.options);
    this.registerApp(this.options.apps || []);
    this.running = true;
    this.hooks.lifecycle.bootstrap.emit(this.options);
    return this;
  }
```
首先还是把传入的options参数深度merge到Garfish默认的options中，接着进行一些插件注册，最后注册了一个名为global-lifecycle的插件，这个插件主要是用来兜底的，因为插件注册是有先后顺序的，先注册的插件，在实行生命钩子方法时，是先执行的，所以global-lifecycle这个插件中，传入的生命周期钩子方法是用户可以自定义传入的，那么最后执行的时候，global-lifecycle拿到的是所有插件中最后的数据，方便调试。
接下来就是触发生命周期中beforeBootstrap，之前所有的插件中，所有beforeBootstrap的钩子都注册到了全局beforeBootstrap中，这个时候进行emit操作。如果用户没有传入自定义plugin(已经定义了beforeBootstrap方法)的话，这里应该是没有可执行方法的，因为框架内置的一些插件中没有该方法。
接下来就是注册app了，我们在主应用中的options中，有个apps的属性，里面注册着所有子应用。看一下registreApp的实现：
```
// packages/core/src/garfish.ts
registerApp(list: interfaces.AppInfo | Array<interfaces.AppInfo>) {
    const currentAdds = {};
    this.hooks.lifecycle.beforeRegisterApp.emit(list);
    if (!Array.isArray(list)) list = [list];

    for (const appInfo of list) {
      assert(appInfo.name, 'Miss app.name.');
      if (!this.appInfos[appInfo.name]) {
        assert(
          appInfo.entry,
          `${appInfo.name} application entry is not url: ${appInfo.entry}`,
        );
        currentAdds[appInfo.name] = appInfo;
        this.appInfos[appInfo.name] = appInfo;
      } else if (__DEV__) {
        warn(`The "${appInfo.name}" app is already registered.`);
      }
    }
    this.hooks.lifecycle.registerApp.emit(currentAdds);
    return this;
  }
```
首先执行生命周期beforeRegisterApp中的方法，然后将apps注册到currentAdds和this.appInfos中，再执行registerApp中的方法。
再返回到run方法中，继续往下执行，就会执行生命周期bootstrap中的方法。我们主要看一下router中的bootstrap方法，看看在启动时做了什么
```
// packages/router/src/index.ts
bootstrap(options: interfaces.Options) {
        let activeApp = null;
        const unmounts: Record<string, Function> = {};
        const { basename } = options;
        const { autoRefreshApp = true, onNotMatchRouter = () => null } =
          Garfish.options;

        async function active(appInfo: interfaces.AppInfo, rootPath: string) {
          const { name, cache = true, active } = appInfo;
          if (active) return active(appInfo, rootPath);
          appInfo.rootPath = rootPath;

          const currentApp = (activeApp = createKey());
          const app = await Garfish.loadApp(appInfo.name, {
            basename: rootPath,
            entry: appInfo.entry,
            cache: true,
            domGetter: appInfo.domGetter,
          });
          app.appInfo.basename = rootPath;

          const call = (app: interfaces.App, isRender: boolean) => {
            if (!app) return;
            const isDes = cache && app.mounted;
            const fn = isRender
              ? app[isDes ? 'show' : 'mount']
              : app[isDes ? 'hide' : 'unmount'];
            return fn.call(app);
          };

          Garfish.apps[name] = app;
          unmounts[name] = () => call(app, false);

          if (currentApp === activeApp) {
            await call(app, true);
          }
        }

        async function deactive(appInfo: interfaces.AppInfo, rootPath: string) {
          activeApp = null;
          const { name, deactive } = appInfo;
          if (deactive) return deactive(appInfo, rootPath);

          const unmount = unmounts[name];
          unmount && unmount();
          delete Garfish.apps[name];

          // Nested scene to remove the current application of nested data
          // To avoid the main application prior to application
          const needToDeleteApps = router.routerConfig.apps.filter((app) => {
            if (appInfo.rootPath === app.basename) return true;
          });
          if (needToDeleteApps.length > 0) {
            needToDeleteApps.forEach((app) => {
              delete Garfish.appInfos[app.name];
              delete Garfish.cacheApps[app.name];
            });
            router.setRouterConfig({
              apps: router.routerConfig.apps.filter((app) => {
                return !needToDeleteApps.some(
                  (needDelete) => app.name === needDelete.name,
                );
              }),
            });
          }
        }

        const apps = Object.values(Garfish.appInfos);

        const appList = apps.filter((app) => {
          if (!app.basename) app.basename = basename;
          return !!app.activeWhen;
        }) as Array<Required<interfaces.AppInfo>>;

        const listenOptions = {
          basename,
          active,
          deactive,
          autoRefreshApp,
          notMatch: onNotMatchRouter,
          apps: appList,
        };
        listenRouterAndReDirect(listenOptions);
      }
```
这个方法可以直接看最后，就是执行了一个listenRouterAndReDirect方法，并传入了listenOptions对象。接着找listenRouterAndReDirect方法的实现：
```
// packages/router/src/context.ts
export const listenRouterAndReDirect = ({
  apps,
  basename,
  autoRefreshApp,
  active,
  deactive,
  notMatch,
}: Options) => {
  // 注册子应用、注册激活、销毁钩子
  registerRouter(apps);

  // 初始化信息
  setRouterConfig({
    basename,
    autoRefreshApp,
    // supportProxy: !!window.Proxy,
    active,
    deactive,
    notMatch,
  });

  // 开始监听路由变化触发、子应用更新。重载默认初始子应用
  listen();
};
```
主要就是注册子应用，初始化配置，最后进行监听。再看listen方法的实现：
```
// packages/router/src/agentRouter.ts
export const listen = () => {
  normalAgent();
  initRedirect();
};
```
然后执行了两个方法normalAgent和initRedirect，继续往下看实现：
```
// packages/router/src/agentRouter.ts
export const normalAgent = () => {
  // By identifying whether have finished listening, if finished listening, listening to the routing changes do not need to hijack the original event
  // Support nested scene
  const addRouterListener = function () {
    window.addEventListener(__GARFISH_BEFORE_ROUTER_EVENT__, function (env) {
      RouterConfig.routerChange && RouterConfig.routerChange(location.pathname);
      linkTo((env as any).detail);
    });
  };

  if (!window[__GARFISH_ROUTER_FLAG__]) {
    // Listen for pushState and replaceState, call linkTo, processing, listen back
    // Rewrite the history API method, triggering events in the call
    const rewrite = function (type: keyof History) {
      const hapi = history[type];
      return function () {
        const urlBefore = window.location.pathname + window.location.hash;
        const stateBefore = history?.state;
        const res = hapi.apply(this as any, arguments);
        const urlAfter = window.location.pathname + window.location.hash;
        const stateAfter = history?.state;

        const e = createEvent(type);
        (e as any).arguments = arguments;

        if (urlBefore !== urlAfter || stateBefore !== stateAfter) {
          if (history.state && history.state === 'object')
            delete history.state[__GARFISH_ROUTER_UPDATE_FLAG__];
          window.dispatchEvent(
            new CustomEvent(__GARFISH_BEFORE_ROUTER_EVENT__, {
              detail: {
                toRouterInfo: {
                  fullPath: urlAfter,
                  query: parseQuery(location.search),
                  path: getPath(RouterConfig.basename!, urlAfter),
                  state: stateAfter,
                },
                fromRouterInfo: {
                  fullPath: urlBefore,
                  query: parseQuery(location.search),
                  path: getPath(RouterConfig.basename!, urlBefore),
                  state: stateBefore,
                },
                eventType: type,
              },
            }),
          );
        }
        // window.dispatchEvent(e);
        return res;
      };
    };

    history.pushState = rewrite('pushState');
    history.replaceState = rewrite('replaceState');

    // Before the collection application sub routing, forward backward routing updates between child application
    window.addEventListener(
      'popstate',
      function (event) {
        // Stop trigger collection function, fire again match rendering
        if (event && typeof event === 'object' && (event as any).garfish)
          return;
        if (history.state && history.state === 'object')
          delete history.state[__GARFISH_ROUTER_UPDATE_FLAG__];
        window.dispatchEvent(
          new CustomEvent(__GARFISH_BEFORE_ROUTER_EVENT__, {
            detail: {
              toRouterInfo: {
                fullPath: location.pathname,
                query: parseQuery(location.search),
                path: getPath(RouterConfig.basename!),
              },
              fromRouterInfo: {
                fullPath: RouterConfig.current!.fullPath,
                path: getPath(
                  RouterConfig.basename!,
                  RouterConfig.current!.path,
                ),
                query: RouterConfig.current!.query,
              },
              eventType: 'popstate',
            },
          }),
        );
      },
      false,
    );

    window[__GARFISH_ROUTER_FLAG__] = true;
  }
  addRouterListener();
};
```

normalAgent方法的实现，其实就是重写了history.pushState和history.replaceState两个方法，并且会触发一个自定义事件__GARFISH_BEFORE_ROUTER_EVENT__，那么addRouterListener其实就是注册了__GARFISH_BEFORE_ROUTER_EVENT__自定义事件。而initRedirect方法就是初始默认化路由的。无论是normalAgent还是initRedirect，最后都会进入linkTo方法：
```
// packages/router/src/linkTo.ts
export const linkTo = async ({
  toRouterInfo,
  fromRouterInfo,
  eventType,
}: {
  toRouterInfo: RouterInfo;
  fromRouterInfo: RouterInfo;
  eventType: keyof History | 'popstate';
}) => {
  const {
    current,
    apps,
    deactive,
    active,
    notMatch,
    beforeEach,
    afterEach,
    autoRefreshApp,
  } = RouterConfig;
  const deactiveApps = current!.matched.filter(
    (appInfo) =>
      !hasActive(
        appInfo.activeWhen,
        getPath(appInfo.basename, location.pathname),
      ),
  );

  // Activate the corresponding application
  const activeApps = apps.filter((appInfo) => {
    return hasActive(
      appInfo.activeWhen,
      getPath(appInfo.basename, location.pathname),
    );
  });

  const needToActive = activeApps.filter(({ name }) => {
    return !current!.matched.some(({ name: cName }) => name === cName);
  });

  // router infos
  const to = {
    ...toRouterInfo,
    matched: needToActive,
  };

  const from = {
    ...fromRouterInfo,
    matched: deactiveApps,
  };

  await toMiddleWare(to, from, beforeEach!);

  // Pause the current application of active state
  if (current!.matched.length > 0) {
    await asyncForEach(
      deactiveApps,
      async (appInfo) =>
        await deactive(appInfo, getPath(appInfo.basename, location.pathname)),
    );
  }

  setRouterConfig({
    current: {
      path: getPath(RouterConfig.basename!),
      fullPath: location.pathname,
      matched: activeApps,
      state: history.state,
      query: parseQuery(location.search),
    },
  });

  // Within the application routing jump, by collecting the routing function for processing.
  // Filtering gar-router popstate hijacking of the router
  // In the switch back and forth in the application is provided through routing push method would trigger application updates
  // application will refresh when autoRefresh configuration to true
  const curState = window.history.state || {};
  if (
    eventType !== 'popstate' &&
    (curState[__GARFISH_ROUTER_UPDATE_FLAG__] || autoRefreshApp)
  ) {
    callCapturedEventListeners(eventType);
  }

  await asyncForEach(needToActive, async (appInfo) => {
    // Function using matches character and routing using string matching characters
    const appRootPath = getAppRootPath(appInfo);
    await active(appInfo, appRootPath);
  });

  if (activeApps.length === 0 && notMatch) notMatch(location.pathname);

  await toMiddleWare(to, from, afterEach!);
};
```
再归纳一下，有两个中间件可以执行，afterEach和beforeEach，就是在active之前和之后的执行时机。然后主要的就是active了，其实active这个方法，是router插件中，bootstrap钩子里的active方法。到目前为止，我们主应用就run起来了，接下来就是通过路由来show或者hide子应用了。
# 路由trigger子应用
接下来点击vue按钮，展示vue子应用。
我们前面已经介绍过了，在normalAgent实现中劫持了history.push方法，那么在进行路由变化时，就会触发自定义事件__GARFISH_BEFORE_ROUTER_EVENT__，然后会再次进入linkTo方法，这个时候needToActive就是true了，会执行active，而active就是router插件中bootstrap的active方法：
```
// packages/router/src/index.ts
async function active(appInfo: interfaces.AppInfo, rootPath: string) {
          const { name, cache = true, active } = appInfo;
          if (active) return active(appInfo, rootPath);
          appInfo.rootPath = rootPath;

          const currentApp = (activeApp = createKey());
          const app = await Garfish.loadApp(appInfo.name, {
            basename: rootPath,
            entry: appInfo.entry,
            cache: true,
            domGetter: appInfo.domGetter,
          });
          app.appInfo.basename = rootPath;

          const call = (app: interfaces.App, isRender: boolean) => {
            if (!app) return;
            const isDes = cache && app.mounted;
            const fn = isRender
              ? app[isDes ? 'show' : 'mount']
              : app[isDes ? 'hide' : 'unmount'];
            return fn.call(app);
          };

          Garfish.apps[name] = app;
          unmounts[name] = () => call(app, false);

          if (currentApp === activeApp) {
            await call(app, true);
          }
        }
```
这个方法中会有个app，这个app是Garfish类中loadApp方法返回的，那我们看一下返回的这个app是啥：
```
// packages/core/src/garfish.ts
async loadApp(
    appName: string,
    optionsOrUrl?: Omit<interfaces.AppInfo, 'name'>,
  ): Promise<interfaces.App | null> {
    assert(appName, 'Miss appName.');
    const appInfo = generateAppOptions(appName, this, optionsOrUrl);

    const asyncLoadProcess = async () => {
      // Return not undefined type data directly to end loading
      const stop = await this.hooks.lifecycle.beforeLoad.emit(appInfo);
      if (stop === false) {
        warn(`Load ${appName} application is terminated by beforeLoad.`);
        return null;
      }
      // Existing cache caching logic
      let appInstance: interfaces.App = null;
      const cacheApp = this.cacheApps[appName];
      if (appInfo.cache && cacheApp) {
        appInstance = cacheApp;
      } else {
        try {
          const [manager, resources, isHtmlMode] = await processAppResources(
            this.loader,
            appInfo,
          );

          appInstance = new App(
            this,
            appInfo,
            manager,
            resources,
            isHtmlMode,
            appInfo.customLoader,
          );

          // The registration hook will automatically remove the duplication
          for (const key in this.plugins) {
            appInstance.hooks.usePlugin(this.plugins[key]);
          }
          if (appInfo.cache) {
            this.cacheApps[appName] = appInstance;
          }
        } catch (e) {
          __DEV__ && warn(e);
          this.hooks.lifecycle.errorLoadApp.emit(e, appInfo);
        }
      }
      await this.hooks.lifecycle.afterLoad.emit(appInfo, appInstance);
      return appInstance;
    };

    if (!this.loading[appName]) {
      this.loading[appName] = asyncLoadProcess().finally(() => {
        this.loading[appName] = null;
      });
    }
    return this.loading[appName];
  }
```
这个函数稍微有点长，我总结一下，this.loading是为了保存当前要加载的app，通过一个asyncLoadProcess返回，这里注意一下，asyncLoadProcess是async函数，没有await，所以返回的是一个Promise对象，而resolve的是appInstance，在asyncLoadProcess中，也是执行了app生命周期中的几个钩子。我们再简单了解一下App类是什么样的，由于代码太多，我们只列个大概，并且说明用途即可：
```
// packages/core/src/module/app.ts
export class App {
  public appId = appId++;
  public display = false;
  public mounted = false;
  public esModule = false;
  public strictIsolation = false;
  public name: string;
  public isHtmlMode: boolean;
  public global: any = window;
  public appContainer: HTMLElement;
  public cjsModules: Record<string, any>;
  public htmlNode: HTMLElement | ShadowRoot;
  public customExports: Record<string, any> = {}; // If you don't want to use the CJS export, can use this
  public sourceList: Array<{ tagName: string; url: string }> = [];
  public appInfo: AppInfo;
  public hooks: interfaces.AppHooks;
  public provider: interfaces.Provider;
  public entryManager: TemplateManager;
  public appPerformance: SubAppObserver;
  /** @deprecated */
  public customLoader: CustomerLoader;

  private active = false;
  private mounting = false;
  private unmounting = false;
  private context: Garfish;
  private resources: interfaces.ResourceModules;
  // Environment variables injected by garfish for linkage with child applications
  private globalEnvVariables: Record<string, any>;
  // es-module save lifeCycle to appGlobalId，appGlobalId in script attr
  private appGlobalId: string;

  constructor(
    context: Garfish,
    appInfo: AppInfo,
    entryManager: TemplateManager,
    resources: interfaces.ResourceModules,
    isHtmlMode: boolean,
    customLoader: CustomerLoader,
  ) {
    // ...
  }

  get rootElement() {
     // ...
  }

  getProvider() {
    // ...
  }

  execScript(
    code: string,
    env: Record<string, any>,
    url?: string,
    options?: { async?: boolean; noEntry?: boolean },
  ) {
     // ...
  }

  // `vm sandbox` can override this method
  runCode(
    code: string,
    env: Record<string, any>,
    url?: string,
    options?: { async?: boolean; noEntry?: boolean },
  ) {
     // ...
  }

  async show() {
     // ...
  }

  hide() {
     // ...
  }

  async mount() {
     // ...
  }

  unmount() {
     // ...
  }

  getExecScriptEnv(noEntry: boolean) {
     // ...
  }

  // Performs js resources provided by the module, finally get the content of the export
  async compileAndRenderContainer() {
     // ...
  }

  private canMount() {
     // ...
  }

  // If asynchronous task encountered in the rendering process, such as triggering the beforeEval before executing code,
  // after the asynchronous task, you need to determine whether the application has been destroyed or in the end state.
  // If in the end state will need to perform the side effects of removing rendering process, adding a mount point to a document,
  // for example, execute code of the environmental effects, and rendering the state in the end.
  private stopMountAndClearEffect() {
     // ...
  }

  // Calls to render do compatible with two different sandbox
  private callRender(provider: interfaces.Provider, isMount: boolean) {
     // ...
  }

  // Call to destroy do compatible with two different sandbox
  private callDestroy(provider: interfaces.Provider, isUnmount: boolean) {
     // ...
  }

  // Create a container node and add in the document flow
  // domGetter Have been dealing with
  private async addContainer() {
    // ...
  }

  private async renderTemplate() {
     // ...
  }

  private async checkAndGetProvider() {
    // ...
  }
}
```
总结起来，这个App类提供的能力如下：
1. 提供静态资源，HTML、CSS、js的结构。
2. 可以在CJS中提取或者推导出子应用的 provider。
3.通过execCode传入模块的CJS规范、require、exports等环境变量实现对外共享
4. 触发渲染：应用相关节点放置在文档流中，依次执行应用脚本，最终渲染功能，执行子应用提供完整的应用独立运行时执行。
5. 触发销毁：执行子应用程序的销毁功能，应用子节点从文档流中移除。
再回到active方法中，最核心的地方是call方法，最后调用了App中的show、mount、hide、unmount方法。show和hide可以理解为之前已经加载过了，就是show和hide一下，mount是要和unmount是挂载和卸载，我们这里主要以mount为例，看一下是如何加载子应用环境并且加载子应用的。
```
// packages/core/src/module/app.ts
async mount() {
    if (!this.canMount()) return false;
    this.hooks.lifecycle.beforeMount.emit(this.appInfo, this, false);

    this.active = true;
    this.mounting = true;
    try {
      // add container and compile js with cjs
      const asyncJsProcess = await this.compileAndRenderContainer();

      // Good provider is set at compile time
      const provider = await this.getProvider();
      // Existing asynchronous functions need to decide whether the application has been unloaded
      if (!this.stopMountAndClearEffect()) return false;

      this.callRender(provider, true);
      this.display = true;
      this.mounted = true;
      this.context.activeApps.push(this);
      this.hooks.lifecycle.afterMount.emit(this.appInfo, this, false);

      await asyncJsProcess;
      if (!this.stopMountAndClearEffect()) return false;
    } catch (e) {
      this.entryManager.DOMApis.removeElement(this.appContainer);
      this.hooks.lifecycle.errorMountApp.emit(e, this.appInfo);
      return false;
    } finally {
      this.mounting = false;
    }
    return true;
  }
```
大概意思就是说，添加了一个子应用容器，拿到子应用资源（主要通过fetch方式获取，继续深挖compileAndRenderContainer就知道了），然后获取子应用export出来的provider，最后执行代码，子应用就成功展示出来了。
再说一下子应用的代码执行，子应用中如果使用了window，那么在子应用接入主应用后，如果不做任何处理，那么两个应用的window是一个，这样就会有逻辑问题，为了解决这个问题，就有了沙箱概念。garfish中提供了两种沙箱机制：vm沙箱和snapshot沙箱。
我们可以看一个简单的vm沙箱原理：
```
const varBox = {};
const fakeWindow = new Proxy(window, {
    get(target, key) {
        return varBox[key] || window[key]
    },
    set(target, key, value) {
        varBox[key] = value;
        return true;
    }
})
const fn = new Function('window', code);
fn(fakeWindow);
```
主要是通过es6中proxy实现的，当然这只是一个原理性代码，实际中还会兼容很多case。其实在最初注册garfish插件的时候，初始化garfish实例的时候，就初始化了vm沙箱和snapshot沙箱。我们前面也说了，在每个插件中都会定义一些生命周期的钩子方法，其实在Garfish.loadApp的时候，就在App上挂载了vmSandbox属性，在后续的子应用执行代码时，环境都是在沙箱中执行。
