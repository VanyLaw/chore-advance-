# 使用堆快照 得到内存使用情况 #
想要分析定位内存泄漏问题，首先我们要去获取 Node.js 进程在发生泄漏时的堆上各个对象和它们间的引用关系，这个保存了堆上各个对象以及其引用关系的文件就是堆快照。V8 引擎提供了一个接口可以让我们很方便地实时获取到堆快照，下面我们介绍三种不同的方法来获取。

## heapdump ##
方法1： 添加到代码。
```
const heapdump = require('heapdump');
const path =  require('path');
setInterval(function() {
  let filename = `${Date.now()}.heapsnapshot`;
  heapdump.writeSnapshot(path.join(__dirname, filename));
}, 30 * 1000);
// 半分钟获取一次
```

方法2： 启动引入了heapdump模块的nodejs进程，通过usr2这个信号量来触发堆快照：

``kill -USR2 <需要获取堆快照的 Node.js 进程 PID>``

这种办法的好处是不需要在代码中植入相关逻辑，而仅在需要的时候 ssh 到服务器上通过信号量获取到堆快照。

## v8-profiler ##
v8-profiler 提供了 transform 流的形式输出堆快照，对于一些比较大的堆快照文件能更好的进行生成处理.
```
const v8Profiler = require('v8-profiler-node8');
const snapshot = v8Profiler.takeSnapshot();
// 获取堆快照数据流
const transform = snapshot.export();
// 流式处理堆快照
transform.on('data', data => console.log(data));
// 数据处理完毕后删除
transform.on('finish', snapshot.delete.bind(snapshot));
```

## Nodejs 性能平台 ##
Node.js 性能平台 目前将堆快照的获取整合进了 runtime 中，只要应用接入平台后，不需要改动业务代码即可在线获取到进程的堆快照以备分析




# 内存泄漏的诱因 #
1. 定义了全局变量
  - （包括未声明的对象【自动挂在global对象】，和全局变量的引用）
2. 闭包
  - 闭包会引用到父级函数中的变量，如果闭包未释放，就会导致内存泄漏。父级函数释放后，会将引用的变量拷贝一份放到闭包里，由子函数的 [[scope]] 引用。
3. 事件监听
  - 对同一个事件重复监听，忘记移除（removeListener），将造成内存泄漏。这种情况很容易在复用对象上添加事件时出现。

  - 例如，Node.js 中 Agent 的 keepAlive 为 true 时，可能造成的内存泄漏。当 Agent keepAlive 为 true 的时候，将会复用之前使用过的 socket，如果在 socket 上添加事件监听，忘记清除的话，因为 socket 的复用，将导致事件重复监听从而产生内存泄漏。

  - 所以，你需要了解添加事件监听的对象的生命周期，并注意自行移除。
4. 其他
  - 比如缓存。在使用缓存的时候，得清楚缓存的对象的多少，如果缓存对象非常多，得做限制最大缓存数量处理。
  - 非常占用 CPU 的代码也会导致内存泄漏，服务器在运行的时候，如果有高 CPU 的同步代码，因为Node.js 是单线程的，所以不能处理处理请求，请求堆积导致内存占用过高。

# 定位内存泄漏 #

## 1. 重现内存泄漏情况 ##
1. 对于只要正常使用就可以重现的内存泄漏，这是很简单的情况只要在测试环境模拟就可以排查了。
2. 对于偶然的内存泄漏，一般会与**特殊的输入**有关系。想稳定重现这种输入是很耗时的过程。如果不能通过 **代码的日志** 定位到这个特殊的输入，那么推荐 **去生产环境打印内存** 快照了。需要注意的是，打印内存快照是很耗 CPU 的操作，可能会对线上业务造成影响。

## 2. 打印内存快照 ##
快照工具推荐使用 heapdump 用来保存内存快照，使用 devtool 来查看内存快照。使用 heapdump 保存内存快照时，只会有 Node.js 环境中的对象。（如果使用 node-inspector 的话，快照中会有前端的变量干扰）。

为了减少正常变量的干扰，可以在打印内存快照之前会调用主动释放内存的 gc() 函数（启动时加上 --expose-gc 参数即可开启）。
```
const heapdump = require('heapdump');
​
const save = function () {
  gc();
  heapdump.writeSnapshot('./' + Date.now() + '.heapsnapshot');
}
```
在打印线上的代码的时候，建议按照内存增长情况来打印快照。推荐打印 3 个内存快照，一个是内存泄漏之前的内存快照，一个是少量测试以后的内存快照，还有一个是多次测试以后的内存快照。

第一个内存快照作为对比，来查看在测试后有哪些对象增长。在内存泄漏不明显的情况下，可以与大量测试以后的内存快照对比，这样能更容易定位。

## 3. 对比内存快照找到泄漏点 ##
通过内存快照找到数量不断增加的对象，找到增加对象是被谁给引用，找到问题代码.
https://github.com/nodejs/node/issues/9268

## 避免泄漏的工具 ##

1. ESLint 检测代码检查非期望的全局变量。

2. 使用闭包的时候，得知道闭包了什么对象，还有引用闭包的对象何时清除闭包。最好可以避免写出复杂的闭包，因为复杂的闭包引起的内存泄漏，如果没有打印内存快照的话，是很难看出来的。

3. 绑定事件的时候，一定得在恰当的时候清除事件。在编写一个类的时候，推荐使用 init 函数对类的事件监听进行绑定和资源申请，然后 destroy 函数对事件和占用资源进行释放。


## 事件侦听器泄漏实例： ##
```
module.exports = app => {
  class HomeController extends app.Controller {
    * demo() {
      	if (ENV === DEVELOPMENT) {
       	//开发环境下操作...
	   } else {
          if (!client) {
              client = Client.create({
                  refreshInterval: 30000,
                  requestTimeout: 5000,
                  urllib: urllib
              })
          }
          client.on('error', err => {
          	//error 处理...
          })

          //其余逻辑处理...
      }
    }
  }
  return HomeController;
};
```
并且在 router.js 中定义的对应这个 controller 的路由如下：
`` app.get(/.*/, 'home.demo'); ``

由于 client 是全局变量，此时用户每访问一次网站首页，都会给 client._events.error 对应的数组增加一个 error 处理函数。一种通用的解决办法是在 error 侦听操作放入 client 的初始化里面：
```
module.exports = app => {
  class HomeController extends app.Controller {
    * demo() {
      	if (ENV === DEVELOPMENT) {
       	//开发环境下操作...
	   } else {
          if (!client) {
              client = Client.create({
                  refreshInterval: 30000,
                  requestTimeout: 5000,
                  urllib: urllib
              })

              client.on('error', err => {
          		//error 处理...
	          })
          }

          //其余逻辑处理...
      }
    }
  }
  return HomeController;
};
```

这样保证全局只有有一个 error 事件侦听器，性能也比较好。还有一种处理方式是每次 controller 处理完成后移除侦听器, 但是这样子比第一种耗费一些额外的性能:
```
module.exports = app => {
  class HomeController extends app.Controller {
    * demo() {
      	if (ENV === DEVELOPMENT) {
       	//开发环境下操作...
	   } else {
          if (!client) {
              client = Client.create({
                  refreshInterval: 30000,
                  requestTimeout: 5000,
                  urllib: urllib
              })
          }
          //定义 error 处理句柄
          const errorHandle = err => {
          		//error 处理...
	       }
          client.on('error', errorHandle);

          //其余逻辑处理...

          //移除 error 侦听器
          client.removeListener('error', errorHandle);
      }
    }
  }
  return HomeController;
}
```
最后一种是 egg 框架推荐的写法，也是本问题的最佳解决办法，像这种进程生命周期只需要一次连接的可以放到 app/extend/application.js 中去 **由框架保证全局单例:**
```
// app/extend/application.js
const CLIENT = Symbol('Application#xxClient');
module.exports = {
  get xxClient() {
    if (!this[CLIENT]) {
      this[CLIENT] = Client.create({});
      // this[CLIENT].on('error', fn);
    }
    return this[CLIENT];
  }
}

// app/controller/home.js
module.exports = app => {
  class HomeController extends app.Controller {
    * demo() {
      this.app.xxClient.xx();
    }
  }
  return HomeController;
};
```

## Cache引发的内存泄漏 ##
Node.js 每个 require 的文件实际上生成了一个 Module 类的实例，并且模块第一次 require 的时候会缓存起来, 那怎么重复引用呢？泄漏点如下：
```
delete require.cache[require.resolve('./Student)];
delete require.cache[require.resolve('./Parent)];
...
```

因为 Node.js 每个 require 的文件实际上生成了一个 Module 类的实例，并且模块第一次 require 的时候会缓存起来：
```
// lib/module.js
var module = new Module(filename, parent);
Module._cache[filename] = module;

// 构造函数
function Module(id, parent) {
  this.id = id;
  this.exports = {};
  this.parent = parent;
  updateChildren(parent, this, false);
  this.filename = null;
  this.loaded = false;
  this.children = [];
}
```
最后就是根据源代码中的映射关系，require.cache === Module._cache:
```
// lib/internal/module.js
function makeRequireFunction() {
  // 我们调用的 require 方法
  function require() {
    //...
  }
  //...
  require.cache = Module._cache;
  return require;
}
```

通过这三段简单的代码可以看到，产生泄漏的代码中 delete require.cache[缓存文件全路径] 确实清理掉了第一次 require 进来时的缓存引用，那为什么会产生泄漏呢？答案在上面的 Module 类的 updateChildren 方法中：
```
// lib/module.js
function updateChildren(parent, child, scan) {
  var children = parent && parent.children;
  if (children && !(scan && children.includes(child)))
    children.push(child);
}
```
可以看到，每次 require 操作，除了自己本身不存在时会缓存自身实例的引用到 Module._cache 中外，还会将自身实例的引用放入 parent.children （如果存在 parent 的话）这个数组中！那么产生泄漏的原因就很明了了：
- delete require.cache 仅仅清除掉了 Module._cache 对文件第一次 require 的实例的引用
- 此时父文件 require 实例的 children 数组中的缓存的引用依旧存在

**下一次 再 require 此文件时，由于 Module._cache 中不存在，会再生成一次文件对应的 Module 实例，同时给其父文件实例的 children 数组再增加一个引用，这样就造成了泄漏。**

## Notice ##
delete require.cache 这个操作是许多希望做 Node.js 热更新的同学喜欢写的代码，根据上面的分析，要完整将这个模块引入去除，需要清除所有引用到此模块的 parent.children 数组里面的引用。

遗憾的是，module.parent 只是保存了**第一次引用该模块的父模块**，而其它引用到此文件的父模块需要开发者自己去构建载入模块间的相互依赖关系，所以在 Node.js 下做热更新并不是一条传统意义上的生路。

# Http KeepAlive导致的内存泄漏（已修复） #
见 https://github.com/nodejs/node/issues/9268 issue：

KeepAlive会复用同一个连接（Socket），会重复给socket实例添加timeout的listener，导致超过最大的监听数目。
> Warning: Possible EventEmitter memory leak detected. 11 timeout listeners added. Use emitter.setMaxListeners() to increase limit

问题代码如下：

```
if(req.timeout){
  socket.once('timeout', ()=>req.emit('timeout'));
}
```


