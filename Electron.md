Electron 应用程序主要分为**主进程**、**渲染进程**两个部分，一个 Electron 应用程序由一个**主进程**（有且只有一个） + 多个**渲染进程**组成。

### **主进程**

功能：桥梁作用，连接操作系统和渲染进程，负责管理所有窗口及其对应的渲染进程。

- 有且只有一个，整个应用入口
- 创建、管理渲染进程
- 控制应用生命周期
- 使用 NodeJS 特性
- 调用操作系统 API
- ...

### **渲染进程**

功能：负责完成渲染页面、接收用户输入、相应用户交互等工作。

- 渲染页面
- 使用部分 Electron 模块 API
- 使用 NodeJS 特性
- 一个应用可存在多个渲染进程
- 控制用户交互逻辑
- 访问 Dom API

### **核心模块归属情况**

![img](https://pic2.zhimg.com/80/v2-c61aa052f9dca40a4c7ed43337dd281d_1440w.webp)

### **electron 优势**

- 上手门槛低
- 开发周期短

### **electron 不足**

- 打包后应用体积过大
- 版本发布过快
- 安全性问题
- 资源消耗较大
- 平台上架难

消息端口是成对创建的，连接的一对消息端口被称为通道。

# 进程间通信 (IPC)

`ipcMain` 从主进程到渲染进程的异步通信。
`ipcRenderer` 从渲染器进程到主进程的异步通信。

## 模式 1：渲染器进程到主进程（单向）

要将单向 IPC 消息从渲染器进程发送到主进程，您可以使用 `ipcRenderer.send` 发送消息，然后使用 `ipcMain.on` 接收。
1、主进程使用 `ipcMain.on` 接收。

```
function handleSetTitle (event, title) {
  const webContents = event.sender
  const win = BrowserWindow.fromWebContents(webContents)
  win.setTitle(title)
}
ipcMain.on('set-title', handleSetTitle)
```

2、通过预加载脚本暴露 `ipcRenderer.send` 向渲染器进程暴露一个全局的 window.electronAPI 变量
在渲染器进程中使用 window.electronAPI.xxx() 函数。

```
const { contextBridge, ipcRenderer } = require('electron')

contextBridge.exposeInMainWorld('electronAPI', {
  xxx: (title) => ipcRenderer.send('set-title', title)
})
```

出于 安全原因，我们不会直接暴露整个 `ipcRenderer.invoke` API。 确保尽可能限制渲染器对 Electron API 的访问。

3、构建渲染器进程 UI

```
window.electronAPI.setTitle('标题')
```

IPC 通道名称 set-title 主进程和渲染进程要保持一致，渲染进程通过 set-title 这个通道名称找到主进程对应的方法

## 模式 2：渲染器进程到主进程（双向）

通过将 `ipcRenderer.invoke` 与 `ipcMain.handle` 搭配使用来完成。
1、主进程中使用 `ipcMain.handle` 监听事件

```
async function handleFileOpen () {
  const { canceled, filePaths } = await dialog.showOpenDialog({})
  if (!canceled) {
    return filePaths[0]
  }
}
ipcMain.handle('dialog:openFile', handleFileOpen)
```

2、通过预加载脚本暴露 `ipcRenderer.invoke`

```
const { contextBridge, ipcRenderer } = require('electron')

contextBridge.exposeInMainWorld('electronAPI', {
  openFile: () => ipcRenderer.invoke('dialog:openFile')
})
```

3、构建渲染器进程 UI

```
const btn = document.getElementById('btn')
const filePathElement = document.getElementById('filePath')

btn.addEventListener('click', async () => {
  const filePath = await window.electronAPI.openFile()
  filePathElement.innerText = filePath
})
```

### Electron 7 之前通过 IPC 进行异步双向通信的旧方法

#### 1、使用 ipcRenderer.send

preload.js (Preload Script)

```
// 您也可以使用 `contextBridge` API
// 将这段代码暴露给渲染器进程
const { ipcRenderer } = require('electron')

ipcRenderer.on('asynchronous-reply', (_event, arg) => {
  console.log(arg) // 在 DevTools 控制台中打印“pong”
})
ipcRenderer.send('asynchronous-message', 'ping')
```

main.js (Main Process)

```
ipcMain.on('asynchronous-message', (event, arg) => {
  console.log(arg) // 在 Node 控制台中打印“ping”
  // 作用如同 `send`，但返回一个消息
  // 到发送原始消息的渲染器
  event.reply('asynchronous-reply', 'pong')
})
```

这种方法有几个缺点：
您需要设置第二个 ipcRenderer.on 监听器来处理渲染器进程中的响应。 使用 invoke，您将获得作为 Promise 返回到原始 API 调用的响应值。
没有显而易见的方法可以将 asynchronous-reply 消息与原始的 asynchronous-message 消息配对。 如果您通过这些通道非常频繁地来回传递消息，则需要添加其他应用代码来单独跟踪每个调用和响应。

#### 2、使用 ipcRenderer.sendSync

ipcRenderer.sendSync API 向主进程发送消息，并 同步 等待响应。
main.js (Main Process)

```
const { ipcMain } = require('electron')
ipcMain.on('synchronous-message', (event, arg) => {
  console.log(arg) // 在 Node 控制台中打印“ping”
  event.returnValue = 'pong'
})
```

preload.js (Preload Script)

```
// 您也可以使用 `contextBridge` API
// 将这段代码暴露给渲染器进程
const { ipcRenderer } = require('electron')

const result = ipcRenderer.sendSync('synchronous-message', 'ping')
console.log(result) // 在 DevTools 控制台中打印“pong”
```

这份代码的结构与 invoke 模型非常相似，但`出于性能原因`，我们建议避免使用此 API。 `它的同步特性意味着它将阻塞渲染器进程`，直到收到回复为止。

## 模式 3：主进程到渲染器进程（单向）

将消息从主进程发送到渲染器进程时，需要指定是哪一个渲染器接收消息。 消息需要通过其 WebContents 实例发送到渲染器进程。 此 WebContents 实例包含一个 send 方法，其使用方式与 ipcRenderer.send 相同。

1、使用 webContents 模块发送消息

```
const mainWindow = new BrowserWindow({
  webPreferences: {
    preload: path.join(__dirname, 'preload.js')
  }
})
mainWindow.webContents.send('update-counter', 1),
```

2、通过预加载脚本暴露 ipcRenderer.on

```
const { contextBridge, ipcRenderer } = require('electron')

contextBridge.exposeInMainWorld('electronAPI', {
  onUpdateCounter: (callback) => ipcRenderer.on('update-counter', (_event, value) => callback(value))
})
```

3、构建渲染器进程 UI

```
const counter = document.getElementById('counter')

window.electronAPI.onUpdateCounter((value) => {
  const oldValue = Number(counter.innerText)
  const newValue = oldValue + value
  counter.innerText = newValue.toString()
})
```

主进程到渲染器进程后，要想从渲染进程再回到主进程，可使用`ipcRenderer.send`与`ipcMain.on`

## 模式 4：渲染器进程到渲染器进程

没有直接的方法可以使用 ipcMain 和 ipcRenderer 模块在 Electron 中的渲染器进程之间发送消息。 为此，您有两种选择：

1、将主进程作为渲染器之间的消息代理。 这需要将消息从一个渲染器发送到主进程，然后主进程将消息转发到另一个渲染器。
2、从主进程将一个 MessagePort 传递到两个渲染器。 这将允许在初始设置后渲染器之间直接进行通信。

# 自动更新

`electron-updater` 中的`autoUpdater`模块
主进程监听`检查更新`事件 ipcMain.on
