国内下载Electron很蛋疼，会遇到卡住不动或者`RequestError: read ECONNRESET`之类的问题。

办法就是使用代理，比如clash默认的代理地址是`http://127.0.0.1:7890`，按照如下步骤安装Electron:

windows:

```shell
set ELECTRON_GET_USE_PROXY=1
set GLOBAL_AGENT_HTTP_PROXY=http://127.0.0.1:7890
npm install -D electron
```

linux or macos:

```shell
export ELECTRON_GET_USE_PROXY=1
export GLOBAL_AGENT_HTTP_PROXY=http://127.0.0.1:7890
npm install -D electron
```

如果仍然安装失败，可以尝试先配置npm的代理：

```shell
npm config set proxy http://127.0.0.1:7890
npm config set https-proxy http://127.0.0.1:7890
```
