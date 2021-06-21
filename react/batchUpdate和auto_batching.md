# react 17 的batchUpdate #
事件处理函数（event handler）内的setState会被batch为一次更新。
在前端框架中实现 Batch Update 的关键在于两个问题：

1. 何时开始一个 queue
2. 何时 flush 这个 queue

React 中的 Batch Update 是通过[「Transaction」](https://github.com/facebook/react/blob/b1768b5a48d1f82e4ef4150e0036c5f846d3758a/src/renderers/shared/stack/reconciler/Transaction.js#L19-L54)实现的。

```
*                       wrappers (injected at creation time)
*                                      +        +
*                                      |        |
*                    +-----------------|--------|--------------+
*                    |                 v        |              |
*                    |      +---------------+   |              |
*                    |   +--|    wrapper1   |---|----+         |
*                    |   |  +---------------+   v    |         |
*                    |   |          +-------------+  |         |
*                    |   |     +----|   wrapper2  |--------+   |
*                    |   |     |    +-------------+  |     |   |
*                    |   |     |                     |     |   |
*                    |   v     v                     v     v   | wrapper
*                    | +---+ +---+   +---------+   +---+ +---+ | invariants
* perform(anyMethod) | |   | |   |   |         |   |   | |   | | maintained
* +----------------->|-|---|-|---|-->|anyMethod|---|---|-|---|-|-------->
*                    | |   | |   |   |         |   |   | |   | |
*                    | |   | |   |   |         |   |   | |   | |
*                    | |   | |   |   |         |   |   | |   | |
*                    | +---+ +---+   +---------+   +---+ +---+ |
*                    |  initialize                    close    |
*                    +-----------------------------------------+
```

Transaction 对一个函数进行包装，让 React 有机会在一个函数运行前后执行特定逻辑，从而完成整个 Batch Update 流程的控制。

简单来说，在 Transaction 的 initialize 阶段，一个 update queue 被创建。在 Transaction 中调用 setState 方法时，状态并不会立即应用，而是被推入到 update queue 中。函数执行结束进入 Transaction 的 close 阶段，update queue 会被 flush，这时新的状态会被应用到组件上并开始后续 Virtual DOM 更新等工作。

与 React 相比 Vue 实现 Batch Update 的方法就要简单很多：直接借助 JavaScript 的 Event Loop。Vue 中 Batch Update 的核心代码只有大约 20 行：

```
// https://github.com/vuejs/vue/blob/dev/src/core/observer/scheduler.js#L122-L148
/**
 * Push a watcher into the watcher queue.
 * Jobs with duplicate IDs will be skipped unless it's
 * pushed when the queue is being flushed.
 */
export function queueWatcher (watcher: Watcher) {
  const id = watcher.id
  if (has[id] == null) {
    has[id] = true
    if (!flushing) {
      queue.push(watcher)
    } else {
      // if already flushing, splice the watcher based on its id
      // if already past its id, it will be run next immediately.
      let i = queue.length - 1
      while (i > index && queue[i].id > watcher.id) {
        i--
      }
      queue.splice(i + 1, 0, watcher)
    }
    // queue the flush
    if (!waiting) {
      waiting = true
      nextTick(flushSchedulerQueue)
    }
  }
}
```

当 model 被修改时，对应的 watcher 会被推入 update queue，与此同时还会在异步队列中添加一个 task 用来 flush 当前 update queue。这样一来，当前 task 中的其他 watcher 会被推入同一个 update queue 中。当前 task 执行结束后，异步队列中的下一个 task 会开始执行，update queue 会被 flush，并进行后续的更新操作。

为了让 flush 动作能在当前 Task 结束后尽可能早的开始，Vue 会优先尝试将任务 micro-task 队列，具体来说，在浏览器环境中 Vue 会优先尝试使用 MutationObserver API 或 Promise，如果两者都不可用，则 fallback 到 setTimeout。

对比两个框架可以发现 React 基于 Transition 实现的 Batch Query 是一个不依赖语言特性的通用模式，因此有更稳定可控的表现，但缺点是无法完全覆盖所有情况，例如setTimeout中的setState就不能批处理。

而 Vue 的实现则对语言特性乃至运行环境有很强的依赖，但可以更好的覆盖各种情况：只要是在同一个 task 中的修改都可以进行 Batch Update 优化。

# react 18 的auto batching #
