## Debug main process

```json
{
    "version": "0.2.0",
    "configurations": [
        {
            "type": "node",
            "name": "ExpenseTracker(Electron Main)",
            "request": "launch",
            "cwd": "${workspaceFolder}",
            "runtimeExecutable": "${workspaceFolder}/node_modules/.bin/electron",
            "windows": {
              "runtimeExecutable": "${workspaceFolder}/node_modules/.bin/electron.cmd"
            },
            "program": "${workspaceFolder}/ExpenseTracker/electron.js",
            "protocol": "inspector"
        }
    ]
}
```

program配置为[[Electron Entry Script]]的路径即可。

## Debug renderer process

prerequisite: [Debugger for Chrome](https://github.com/Microsoft/vscode-chrome-debug)

```json
{
    "version": "0.2.0",
    "configurations": [
        {
            "type": "chrome",
            "name": "ExpenseTracker(Electron Renderer)",
            "request": "launch",
            "runtimeExecutable": "${workspaceFolder}/node_modules/.bin/electron",
            "windows": {
              "runtimeExecutable": "${workspaceFolder}/node_modules/.bin/electron.cmd"
            },
            "runtimeArgs": [
              "${workspaceFolder}/ExpenseTracker/electron.js",
              "--remote-debugging-port=9222"
            ],
            "webRoot": "${workspaceFolder}"
        },
    ]
}
```

runtimeArgs的首个参数配置为[[Electron Entry Script]]的路径即可。
