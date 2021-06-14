# (第三方 web 应用进程隔离)[https://imwangfu.com/blog/iframe-site-isolation.html] #
通常而言，我们可以使用 web worker 进行多线程计算，线程可以执行任务而不干扰用户界面。但web worker 不能直接操作DOM节点，通常会用于做一些 UI 无关的计算或者执行 I/O 操作。对于一个嵌入 web 内的独立第三方应用，我们有没有办法实现进程级别的隔离呢？

在 web 中，我们能够想到的方式是 iframe, 但长久以来 iframe 被各种诟病和误解。
其无非是以下几个原因：

- iframe 会阻塞主页面 onload 事件
- iframe 需要加载完整的页面，包括所有的css、js 依赖，初始化速度慢。
- iframe 内的页面需要实现所有的渲染、交互等逻辑，逻辑太重，消耗性能。

第一点，我们可以通过动态创建 iframe 来规避掉。后面两点纯粹是误解，iframe 的使用场景本身就是为嵌入独立 web 应用而生，它也只应该在这个场景使用。

如今，对于自己可控的动态内容，我们理所当然通过 ajax 去获取，但对于嵌入不受控的独立第三方应用，iframe 才是它真正的战场。

虽然可以独立嵌入第三方app，做到css、js 等上下文完全隔离，传统 iframe 仍然有一个严重的问题，它和父页面共享一个进程，这就容易造成一个问题，对于性能低下的 iframe 应用会直接卡死当前页面。在 chrome 67 以上浏览器中提供了一个 site-isolation 特性，让独立进程渲染成为可能。

**这一特性的初衷是为了保护浏览器网页不受其他站点攻击，以沙盒的形式进行站点隔离，其中包括网页中嵌入的 iframe。** （同源策略是第一道防线）

site-isolation使每个进程都在一个沙箱中运行，限制了进程所能做的事情。【防止Meltdown and Spectre等】

触发site-isolation的前提是嵌入的iframe和父页面不同域名，浏览器会开启独立进程渲染iframe内容，这一浏览器技术叫 out-of-process iframe。
!()[./image/1593329475_29_w1007_h580.6558d6c8.png]

简单看下效果： 父页面
```
// 同域
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>iframe test</title>
</head>
<body>
    hello
    <iframe src="iframe.html"></iframe>
</body>
<script>
 var rAF = function () {
    return (
        window.requestAnimationFrame ||
        window.webkitRequestAnimationFrame ||
        function (callback) {
            window.setTimeout(callback, 1000 / 60);
        }
    );
}();
  
var frame = 0;
var allFrameCount = 0;
var lastTime = Date.now();
var lastFameTime = Date.now();
  
var loop = function () {
    var now = Date.now();
    var fs = (now - lastFameTime);
    var fps = Math.round(1000 / fs);
  
    lastFameTime = now;
    allFrameCount++;
    frame++;
  
    if (now > 200 + lastTime) {
        var fps = Math.round((frame * 1000) / (now - lastTime));
        console.log(`parent-FPS：`, fps);
        frame = 0;
        lastTime = now;
    };
  
    rAF(loop);
}
 
loop();
</script>
</html>
```
我们在 iframe 加入了定时进行性能消耗的操作，然后在父页面打印 fps。
!()[./image/fps-parent.png]

可以看到，子页面卡顿时，父页面会被阻塞掉。

现在把 iframe 域名切换成和父页面不同的地址，但逻辑相同：
```
<body>
    hello
    <iframe src="http://127.0.0.1:8081/iframe.html"></iframe>
</body>
```
父页面 fps 完全没有受到影响：
!()[./image/1593327168_7_w197_h262.5c4e8ada.png]

通过 chrome 进行查询：**chrome://process-internals/#web-contents**.
可以看到，不同域名的 iframe 浏览器使用的进程 id 是不一样的。

## 没有site-isolation前可能遇到的问题 ##
1. 被破坏的渲染进程
2. 跨站脚本攻击UXSS
3. 侧通道攻击 side channel attack（spectre和meltdown）
https://www.chromium.org/Home/chromium-security/site-isolation

保护措施如下：
1. 跨站点文档总是被放到不同的进程中，无论导航是在当前选项卡，新选项卡，还是在iframe(即一个网页嵌入到另一个网页中)。
2. 跨站点数据(特别是HTML、XML、JSON和PDF文件)不会被发送到网页的进程中，除非服务器表示允许(使用CORS)。
3. 浏览器进程中的安全检查可以检测并终止行为不正常的渲染进程。

具体设计见：
https://www.chromium.org/developers/design-documents/site-isolation


