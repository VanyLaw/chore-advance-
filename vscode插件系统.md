## vscode 插件的强隔离 ##
- 独立进程：VSCode plugin 代码运行在只属于自己的独立 Extension Host 宿主进程里
- 逻辑与视图隔离：插件完全无法访问 DOM 以及操作 UI，插件只能响应 VSCode Core 暴露的扩展点
- 视图扩展能力非常弱：VSCode 有非常稳定的交互与视觉设计，提供给插件的 UI 上的洞（component slot）非常少且稳定
- 只能使用限制的组件来扩展：VSCode 对视图扩展的能力限制非常强，洞里面的 UI 是并不能随意绘制的，只能使用一些官方提供的内置组件，比如 TreeView 之类

## 插件API的注入 ##
插件开发者调用 core 能力时需要引入名为 vscode 的 npm 模块
```
import * as vscode from 'vscode';
```
而实际上这只是一个 vscode.d.ts 类型声明文件，它声明了所有插件可用的 API 类型。
这些 API 的具体实现在 src/vs/workbench/api/common/extHost.api.impl.ts createApiFactoryAndRegisterActors

那么具体这些 API 在 plugin 执行上下文是何时注入的呢？其实是在插件 import 语句执行的时候动了手脚。

```
// /src/vs/workbench/api/common/extHostRequireInterceptor.ts

class VSCodeNodeModuleFactory implements INodeModuleFactory {
    public load(_request: string, parent: URI): any {

        // get extension id from filename and api for extension
        const ext = this._extensionPaths.findSubstr(parent.fsPath);
        if (ext) {
            let apiImpl = this._extApiImpl.get(ExtensionIdentifier.toKey(ext.identifier));
            if (!apiImpl) {
                apiImpl = this._apiFactory(ext, this._extensionRegistry, this._configProvider);
                this._extApiImpl.set(ExtensionIdentifier.toKey(ext.identifier), apiImpl);
            }
            return apiImpl;
        }

        // fall back to a default implementation
        if (!this._defaultApiImpl) {
            let extensionPathsPretty = '';
            this._extensionPaths.forEach((value, index) => extensionPathsPretty += `\t${index} -> ${value.identifier.value}\n`);
            this._logService.warn(`Could not identify extension for 'vscode' require call from ${parent.fsPath}. These are the extension path mappings: \n${extensionPathsPretty}`);
            this._defaultApiImpl = this._apiFactory(nullExtensionDescription, this._extensionRegistry, this._configProvider);
        }
        return this._defaultApiImpl;
    }
}
```

## 插件的开发 ##
插件核心是配置文件，一些关键的配置如：main，主文件入口，比如导出一个 activate 方法，可以接受 ctx 做一些事情。
```
// package.json:
//  "main": "./src/extension.js",
// extension.js
const vscode = require("vscode");

function activate(context) {
  console.log('Congratulations, your extension "helloworld" is now active!');

  let disposable = vscode.commands.registerCommand(
    "extension.helloWorld",
    function() {
      vscode.window.showInformationMessage("Hello World!");
    }
  );

  context.subscriptions.push(disposable);
}

exports.activate = activate;
```
调用activate的时机是一些event触发。可在package.json中的activationEvents配置。
```
onLanguage：包含该语言类型的文件被打开
onLanguage:json
onCommand：某个命令
onCommand:extension.sayHello
onDebug：开始调试
onDebugInitialConfigurations
onDebugResolve
workspaceContains：有匹配规则的文件被打开
workspaceContains:**/.editorconfig
onFileSystem：打开某个特殊协议的文件
onFileSystem:sftp
onView：某个 id 的视图被显示
onView:nodeDependencies
onUri：向操作系统注册的 schema
vscode://vscode.git/init
onWebviewPanel：某种 viewType 的 webview 打开时
onWebviewPanel:catCoding
*：启动就立即打开
```

