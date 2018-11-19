---
title: "Electron 单实例应用程序实现要点"
date: 2018-11-19T21:45:02+08:00
categories:
- Electron
tags:
- Electron
- Typescript
- MacOS
---

## 实现方式

通过 Electron app 对象的方法 `app.makeSingleInstance` 实现, 它的原型如下

```js
/**
 * app.makeSingleInstance
 *
 * @param argv: 第二个运行实例的命令行参数
 * @param workingDirectory: 第二个运行实例的工作目录
 * @return true 表示这是第二个实例, false 表示这是第一个实例
 */
function makeSingleInstance((argv: string[], workingDirectory: String)): boolean;
```

使用方法

```js
const {app} = require('electron')
let myWindow = null

const isSecondInstance = app.makeSingleInstance((commandLine, workingDirectory) => {
  // ⚠️ 当启动第二个实例的时候, 第一个实例进程中的这个方法会被调用
  // ⚠️ Electron 会保证这个回调函数会在(第一个实例进程的) app.ready 之后触发

  // 在这里第一个实例进程可以获取第二个实例的命令行参数, 从而进行处理
  // 通常我们会聚焦主窗口
  if (myWindow) {
    if (myWindow.isMinimized()) myWindow.restore()
    myWindow.focus()
  }
})

// 为了保证单实例, 退出第二个实例
if (isSecondInstance) {
  app.quit()
}

app.on('ready', () => {
})
```

## 需要注意 (`macOS`)

在 `macOS` 上操作系统会自动保证单实例, 所以当使用以下方式启动第二个实例时, ⚠️ `app.makeSingleInstance` 的回调函数不会触发

1. 在 Finder 中打开
1. 在命令行使用 `open xxx/xxx.app` 方式打开
1. 打开关联的文件
1. 打开关联的 url 协议, 例如 `thunder://...`, `vscode://...`

情况 `3` 会触发 [Electron app 对象的 'open-file' 事件](https://electronjs.org/docs/api/app#%E4%BA%8B%E4%BB%B6-open-file-macos)
情况 `4` 会触发 [Electron app 对象的 'open-url' 事件](https://electronjs.org/docs/api/app#%E4%BA%8B%E4%BB%B6-open-url-macos)

要解决这个问题, 可以考虑以下方案:

1. 命令行用 `open -n xxx/xxx.app` 打开, `-n` 表示允许多开
1. 命令行用 app 里面的可执行文件 `xxx/xxx.app/Contents/MacOS/xxx` 来打开

⚠️ 还有一种特殊的情况, 在两个不同的路径有相同的一个 app, 比如: `~/Applications/Editor.app` 和 `~/Download/Editor.app`. 假如先启动的是前者, 然后双击启动后者, 这种情况不会触发 `macOS` 系统的单实例机制, 而会触发 `app.makeSingleInstance` 的回调


## API 变化

在 [3.0.x 版本](https://github.com/electron/electron/blob/3-0-x-appcontainer-trials/docs/api/app.md) 引入了新的 API: `app.requestSingleInstanceLock()`, [`app.makeSingleInstance` 计划在 4.0 版本废弃](https://github.com/electron/electron/blob/master/docs/api/breaking-changes.md)

### `app.makeSingleInstance`

```js
// Deprecated
app.makeSingleInstance(function (argv, cwd) {

})
// Replace with
app.requestSingleInstanceLock()
app.on('second-instance', function (event, argv, cwd) {

})
```

### `app.releaseSingleInstance`

```js
// Deprecated
app.releaseSingleInstance()
// Replace with
app.releaseSingleInstanceLock()
```

从上面的代码可以看出, 新 API 的用法和之前大致相同, 只是回调改为 app 的一个事件 `'second-instance'`.
