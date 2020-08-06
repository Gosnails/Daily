Electron是由Github开发，用HTML，CSS和JavaScript来构建跨平台桌面应用程序的一个开源库。 Electron通过将Chromium和Node.js合并到同一个运行时环境中，并将其打包为Mac，Windows和Linux系统下的应用来实现这一目的。

### 添加启动动画

由于Electron第一次启动比较慢,需要一些启动动画提高下用户体验。在主进程添加以下代码

```javascript
const electron = require('electron')
const app = electron.app
const BrowserWindow = electron.BrowserWindow
let mainWindow

function createWindow(callback) {
    mainWindow = new BrowserWindow({
        width: 1400,
        height: 900,
        show: false
    })

    mainWindow.once('ready-to-show', () => {
      if (callback){
        callback();
      }
        mainWindow.show()
    })
    mainWindow.loadURL(`file://${__dirname}/build/index.html`);
}
app.on('ready', () => {
    const useH = parseInt(0.865 * electron.screen.getPrimaryDisplay().workAreaSize.height);
    const useW = parseInt(0.625 * electron.screen.getPrimaryDisplay().workAreaSize.width);

    let logoWindow = new BrowserWindow({
        width: useW,
        height: useH,
        transparent: true,
        frame: false,
        center: true,
        show: false
    });
    logoWindow.loadURL(`file://${__dirname}/build/logo/logo.html`);
    logoWindow.once('ready-to-show', () => {
        logoWindow.show();
    });
    const closeLogoWindow = () => {
        logoWindow.close();
    };
    createWindow(closeLogoWindow);
})
```


### 实现自动更新

使用 `electron-builder` 结合 `electron-updater` 实现自动更新。

配置package.json文件

```json
"build": {
    "publish": [
      {
        "provider": "generic",
        "url": "http://ossactivity.tongyishidai.com/"
      }
    ]
}
```

主进程添加自动更新检测和事件监听：

```javascript
import { autoUpdater } from "electron-updater"
import { ipcMain } from "electron"

function updateHandle() {
  let message = {
    error: '检查更新出错',
    checking: '正在检查更新……',
    updateAva: '检测到新版本，正在下载……',
    updateNotAva: '现在使用的就是最新版本，不用更新',
  };

  autoUpdater.setFeedURL('http://ossactivity.tongyishidai.com/');
  autoUpdater.on('error', function (error) {
    console.log(error)
    sendUpdateMessage(message.error)
  });
  autoUpdater.on('checking-for-update', function () {
    sendUpdateMessage(message.checking)
  });
  autoUpdater.on('update-available', function (info) {
    sendUpdateMessage(message.updateAva)
  });
  autoUpdater.on('update-not-available', function (info) {
    sendUpdateMessage(message.updateNotAva)
  });

  // 更新下载进度事件
  autoUpdater.on('download-progress', function (progressObj) {
    mainWindow.webContents.send('downloadProgress', progressObj)
  })
  autoUpdater.on('update-downloaded', function (event, releaseNotes, releaseName, releaseDate, updateUrl, quitAndUpdate) {

    ipcMain.on('isUpdateNow', (e, arg) => {
      console.log(arguments);
      console.log("开始更新");
      //some code here to handle event
      autoUpdater.quitAndInstall();
    });

    mainWindow.webContents.send('isUpdateNow')
  });

  ipcMain.on("checkForUpdate",()=> {
      //执行自动更新检查
      autoUpdater.checkForUpdates();
  })
}

// 通过main进程发送事件给renderer进程，提示更新信息
function sendUpdateMessage(text) {
  mainWindow.webContents.send('message', text)
}
```
> 注意：
> * 在添加自动更新检测和事件监听之后，在主进程createWindow中需要调用一下updateHandle()。
> * 这个autoUpdater不是electron中的autoUpdater，是electron-updater的autoUpdater。(这里曾报错曾纠结一整天)


在视图（View）层中触发自动更新，并添加自动更新事件的监听。

触发自动更新：
```javascript
  ipcRenderer.send("checkForUpdate");
```

监听自动更新事件：
```javascript
    ipcRenderer.on("message", (event, text) => {
      console.log(text)
    });
    ipcRenderer.on("downloadProgress", (event, progressObj)=> {
      console.log(progressObj.percent)
      this.setState({
        downloadPercent: parseInt(progressObj.percent) || 0
      })
    });
    ipcRenderer.on("isUpdateNow", () => {
        ipcRenderer.send("isUpdateNow");
    });
```

为避免多次切换页面造成监听的滥用，切换页面前必须移除监听事件：

```javascript
ipcRenderer.removeAll(["message", "downloadProgress", "isUpdateNow"]);
```

> 参考地址 https://segmentfault.com/a/1190000012904543

### 使用 `electron-builder` 打包exe安装包

package.json文件配置如下

```javascript
{
"build": {
    "productName": "CubeScratch",
    "appId": "org.develar.CubeScratch",
    "publish": [
      {
        "provider": "generic",
        "url": "http://ossactivity.tongyishidai.com/"
      }
    ],
    "win": {
      "icon": "icon.ico",
      "artifactName": "${productName}_Setup_${version}.${ext}",
      "target": [
        "nsis"
      ]
    },
    "mac": {
      "icon": "icon.icns"
    },
    "nsis": {
      "oneClick": false,
      "allowElevation": true,
      "allowToChangeInstallationDirectory": true,
      "installerIcon": "icon.ico",
      "uninstallerIcon": "icon.ico",
      "installerHeaderIcon": "icon.ico",
      "createDesktopShortcut": true,
      "createStartMenuShortcut": true,
      "shortcutName": "CubeScratch"
    },
    "directories": {
      "buildResources": "resources",
      "output": "release"
    },
    "dmg": {
      "contents": [
        {
          "x": 410,
          "y": 150,
          "type": "link",
          "path": "/Applications"
        },
        {
          "x": 130,
          "y": 150,
          "type": "file"
        }
      ]
    }
  }
}

```

> 参考地址 https://juejin.im/post/5bc53aade51d453df0447927

### windows平台 `serialport` 串口编译

如果需要serialport作为Electron项目的依赖项，则必须针对项目使用的Electron版本对其进行编译。

对于大多数最常见的用例（标准处理器平台上的Linux，Mac，Windows），我们使用prebuild来编译和发布库的二进制文件。
使用nodejs进行编译node-gyp需要使用Python 2.x，因此请确保已安装它，并且在所有操作系统的路径中。Python 3.x无法正常工作。

安装windows构建工具和配置想省时省力请选择以下方案：

```
npm install --global --production windows-build-tools
```

上边命令决定串口编译是否成功。安装过程非常缓慢,安装完成就等于串口编译成功了99%。

手动安装工具和配置请看https://github.com/nodejs/node-gyp/

接下来用 electron-rebuild 包重建模块以适配 Electron。这个包可以自动识别当前 Electron 版本，为你的应用自动完成下载 headers、重新编译原生模块等步骤。

```json
"scripts": {
    "rebuild": "electron-rebuild"
}
```

执行以下命令完成串口编译

```
npm run rebuild
```

> 注意：
> * 编译过的串口不同系统不可通用，需在各平台重新编译
> * windows系统最好是正版，或净化版。否正很有可能安装失败。
