## Chrome

常用chrome来debug web项目，尽管[msjsdiag.debugger-for-chrome](https://marketplace.visualstudio.com/items?itemName=msjsdiag.debugger-for-chrome)已经deprecated。后面如果这个拓展确实不能用了，可以尝试使用[codemooseus.vscode-devtools-for-chrome](https://marketplace.visualstudio.com/items?itemName=codemooseus.vscode-devtools-for-chrome)。

### 依赖拓展

- [msjsdiag.debugger-for-chrome](https://marketplace.visualstudio.com/items?itemName=msjsdiag.debugger-for-chrome)
- [ritwickdey.LiveServer](https://marketplace.visualstudio.com/items?itemName=ritwickdey.LiveServer)

### launch.json

可以有2种配置，一种是直接用浏览器开启本地的html启动调试，另一种是通过LiveServer启动本地服务器然后通过url启动调试。

#### 开启本地html启动调试

```json
{
    "version": "0.2.0",
    "configurations": [
        {
            "type": "pwa-chrome",
            "request": "launch",
            "name": "Breakout",
            "file": "${workspaceFolder}/breakout-game/index.html"
        }
    ]
}
```

这个就很简单了，file指向html的本地路径即可。

#### 通过LiveServer调试

首先在vscode的右下角启动LiveServer，并留意端口号，通常都是5500。

启动后，通过下面这个url，就可以访问本地服务器的根目录:

```
http://localhost:5500
```

如果`index.html`不在根目录下，可以通过下面这个url访问指定的`index.html`:

```
http://localhost:5500/YourAwesomeDirName
```

然后launch.json配置要debug的url即可。

```json
{
    "version": "0.2.0",
    "configurations": [
        {
            "type": "chrome",
            "request": "launch",
            "name": "Launch Chrome against localhost",
            "url": "http://localhost:5500/YourAwesomeDirName",
            "webRoot": "${workspaceFolder}"
        }
    ]
}
```

## settings.json

settings.json种可以配置一些js的配置，例如:

```json
{
    "javascript.inlayHints.parameterNames.enabled": "all",  // 是否在函数调用时显示参数名字
    "javascript.inlayHints.variableTypes.enabled": false  // 是否显示变量类型
}
```
