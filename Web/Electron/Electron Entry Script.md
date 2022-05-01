入口脚本基本是一个固定的写法，例如下面的脚本:

```js
// @ts-check

const electron = require('electron');
const app = electron.app;
const BrowserWindow = electron.BrowserWindow;

let mainWindow = null;

function createWindow() {
    mainWindow = new BrowserWindow({
        width: 800,
        height: 600,
    });

    mainWindow.loadFile('index.html');

    mainWindow.on('closed', () => {
        mainWindow = null;
    });
}

app.whenReady().then(() => {
    createWindow();

    app.on('activate', () => {
        if (BrowserWindow.getAllWindows().length === 0) {
            createWindow();
        }
    });
});

app.on('window-all-closed', () => {
    if (process.platform !== 'darwin') {
        app.quit();
    }
});

```

脚本解释：

- app的ready事件发布时从能创建BrowserWindow
- BrowserWindow可以用loadFile加载一个本地html，或者用loadURL加载一个URL也可以，当然URL也可以是`file://${__dirname}/index.html`
- window-all-closed事件时，macos可以不调`app.quit()`因为macos可以保留这个进程而没有窗口，用户可以自己用command + Q来quit进程，同样的当进程被保留时，当用户在dock再次点击对应app的icon时，app会发布activate事件，在这个事件下再次创建主窗口即可a
