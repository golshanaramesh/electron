# Breaking Chanbefore the change is made.

### Types of Breaking Changes

This document uses the following convention to categorize breaking changes:

* **API Changed:** An API was changed in such a way that code that has not been updated is guaranteed to throw an exception.
* **Behavior Changed:** The behavior of Electron has changed, but not in such a way that an exception will necessarily be thrown.
* **Default Changed:** Code depending on the old default may break, not necessarily throwing an exception. The old behavior can be restored by explicitly specifying the value.
* **Deprecated:** An API was marked as deprecated. The API will continue to function, but will emit a deprecation warning, and will be removed in a future release.
* **Removed:** An API or feature was removed, and is no longer supported by Electron.

## Planned Breaking API Changes (34.0)

### Deprecated: `level`, `message`, `line`, and `sourceId` arguments in `console-message` event on `WebContents`

The `console-message` event on `WebContents` has been updated to provide details on the `Event`
argument.

```js
// Deprecated
webContents.on('console-message', (event, level, message, line, sourceId) => {})

// Replace with:
webContents.on('console-message', ({ level, message, lineNumber, sourceId, frame }) => {})
```

Additionally, `level` is now a string with possible values of `info`, `warning`, `error`, and `debug`.

## Planned Breaking API Changes (33.0)

### Behavior Changed: frame properties may retrieve detached WebFrameMain instances or none at all

APIs which provide access to a `WebFrameMain` instance may return an instance
with `frame.detached` set to `true`, or possibly return `null`.

When a frame performs a cross-origin navigation, it enters into a detached state
in which it's no longer attached to the page. In this state, it may be running
[unloa
handlers prior to being deleted. In the event of an IPC sent during this state,
`frame.detached` will be set to `true` with the frame being destroyed shortly
thereafter.

When receiving an event, it's important to access WebFrameMain properties
immediately upon being received. Otherwise, it's not guaranteed to point to the
same webpage as when received. To avoid misaligned expectations, Electron will
return `null` in the case of late access where the webpage has changed.

```ts
ipcMain.on('unload-event', (event) => {
  event.senderFrame; // ✅ accessed immediately
});

ipcMain.on('unload-event', async (event) => {
  await crossOriginNavigationPromise;
  event.senderFrame; // ❌ returns `null` due to late access
});
```

### Behavior Changed: custom protocol URL handling on Windows

Due to changes made in Chromium to support [Non-Special Schem custom protocol URLs that use Windows file paths will no longer work correctly with the deprecated `protocol.registerFileProtocol` and the `baseURLForDataURL` property on `BrowserWindow.loadURL`, `WebContents.loadURL`, and `<webview>.loadURL`.  `protocol.handle` will also not work with these types of URLs but this is not a change since it has always worked that way.

```js
// No longer works
protocol.registerFileProtocol('other', () => {
  callback({ filePath: '/path/to/my/file' })
})

const mainWindow = new BrowserWindow()
mainWindow.loadURL('data:text/html,<script src="loaded-from-dataurl.js"></script>', { baseURLForDataURL: 'other://C:\\myapp' })
mainWindow.loadURL('other://C:\\myapp\\index.html')

// Replace with
const path = require('node:path')
const nodeUrl = require('node:url')
protocol.handle(other, (req) => {
  const srcPath = 'C:\\myapp\\'
  const reqURL = new URL(req.url)
  return net.fetch(nodeUrl.pathToFileURL(path.join(srcPath, reqURL.pathname)).toString())
})

mainWindow.loadURL('data:text/html,<script src="loaded-from-dataurl.js"></script>', { baseURLForDataURL: 'other://' })
mainWindow.loadURL('other://index.html')
```

### Behavior Changed: menu bar will be hidden during fullscreen on Windows

This brings the behavior to parity with Linux. Prior behavior: Menu bar is still visible during fullscreen on Windows. New behavior: Menu bar is hidden during fullscreen on Windows.

### Behavior Changed: `webContents` property on `login` on `app`

The `webContents` property in the `login` event from `app` will be `null`
when the event is triggered for re`respondToAuthRequestsFromMainProcess` option.

### Deprecated: `textured` option in `BrowserWindowConstructorOption.type`

The `textured` option of `type` in `BrowserWindowConstructorOptions` has been deprecated with no replacement. This option relied on thd style mask on macOS, which has been deprecated with no alternative.

### Removed: macOS 10.15 support

macOS 10.15 (Catalina) is no longer supported by [Chromium

Older versions of Electron will continue to run on Catalina, but macOS 11 (Big Sur)
or later will be required to run Electron v33.0.0 and higher.

### Deprecated: `systemPreferences.accessibilityDisplayShouldReduceTransparency`

The `systemPreferences.accessibilityDisplayShouldReduceTransparency` property is now deprecated in favor of the new `nativeTheme.prefersReducedTransparency`, which provides identical information and works cross-platform.

```js
// Deprecated
const shouldReduceTransparency = systemPreferences.accessibilityDisplayShouldReduceTransparency

// Replace with:
const prefersReducedTransparency = nativeTheme.prefersReducedTransparency
```

## Planned Breaking API Changes (32.0)

### Removed: `File.path`

The nonstandard `path` property of the Web `File` object was added in an early version of Electron as a convenience method for working with native files when doing everything in the renderer was more common. However, it represents a deviation from the standard and poses a minor security risk as well, so beginning in Electron 32.0 it has been removed in favor of the 

```js
// Before (renderer)

const file = document.querySelector('input[type=file]')
alert(`Uploaded file path was: ${file.path}`)
```

```js
// After (renderer)

const file = document.querySelector('input[type=file]')
electron.showFilePath(file)

// (preload)
const { contextBridge, webUtils } = require('electron')

contextBridge.exposeInMainWorld('electron', {
  showFilePath (file) {
    // It's best not to expose the full file path to the web content if
    // possible.
    const path = webUtils.getPathForFile(file)
    alert(`Uploaded file path was: ${path}`)
  }
})
```

### Deprecated: `clearHistory`, `canGoBack`, `goBack`, `canGoForward`, `goForward`, `goToIndex`, `canGoToOffset`, `goToOffset` on `WebContents`

The navigation-related APIs are now deprecated.

These APIs have been moved to the `navigationHistory` property of `WebContents` to provide a more structured and intuitive interface for managing navigation history.

```js
// Deprecated
win.webContents.clearHistory()
win.webContents.canGoBack()
win.webContents.goBack()
win.webContents.canGoForward()
win.webContents.goForward()
win.webContents.goToIndex(index)
win.webContents.canGoToOffset()
win.webContents.goToOffset(index)

// Replace with
win.webContents.navigationHistory.clear()
win.webContents.navigationHistory.canGoBack()
win.webContents.navigationHistory.goBack()
win.webContents.navigationHistory.canGoForward()
win.webContents.navigationHistory.goForward()
win.webContents.navigationHistory.canGoToOffset()
win.webContents.navigationHistory.goToOffset(index)
```

## Planned Breaking API Changes (31.0)

### Removed: `WebSQL` support

Chromium has removed support for WebSQL upstream, transitioning it to Android only. S
for more information.

### Behavior Changed: `nativeImage.toDataURL` will preserve PNG colorspace

PNG decoder implementation has been changed to preserve colorspace data, the
encoded data returned from this function now matches it.

See for more information.

### Behavior Changed: `window.flashFrame(bool)` will flash dock icon continuously on macOS

This brings the behavior to parity with Windows and Linux. Prior behavior: The first `flashFrame(true)` bounces the dock icon only once (using the level) and `flashFrame(false)` does nothing. New behavior: Flash continuously until `flashFrame(false)` is called. This uses the [NSCriticalRequest level instead. To explicitly use `NSInformationalRequest` to cause a single dock icon bounce, it is still possible to us

## Planned Breaking API Changes (30.0)

### Behavior Changed: cross-origin iframes now use Permission Policy to access features

Cross-origin iframes must now specify features available to a given `iframe` via the `allow`
attribute in order to access them.

See [documentation for
more information.

### Removed: The `--disable-color-correct-rendering` switch

This switch was never formally documented but it's removal is being noted here regardless. Chromium itself now has better support for color spaces so this flag should not be needed.

### Behavior Changed: `BrowserView.setAutoResize` behavior on macOS

In Electron 30, BrowserView is now a wrapper aro API.

Previously, the `setAutoResize` function of the `BrowserView` API was backed by [autor on macOS, and by a custom algorithm on Windows and Linux.
For simple use cases such as making a BrowserView fill the entire window, the behavior of these two approaches was identical.
However, in more advanced cases, BrowserViews would be autoresized differently on macOS than they would be on other platforms, as the custom resizing algorithm for Windows and Linux did not perfectly match the behavior of macOS's autoresizing API.
The autoresizing behavior is now standardized across all platforms.

If your app uses `BrowserView.setAutoResize` to do anything more complex than making a BrowserView fill the entire window, it's likely you already had custom logic in place to handle this difference in behavior on macOS.
If so, that logic will no longer be needed in Electron 30 as autoresizing behavior is consistent.

### Deprecated: `BrowserViave
also been deprecated:

```js
BrowserWindow.fromBrowserView(browserView)
win.setBrowserView(browserView)
win.getBrowserView()
win.addBrowserView(browserView)
win.removeBrowserView(browserView)
win.setTopBrowserView(browserView)
win.getBrowserViews()
```

### Removed: `params.inputFormType` property on `context-menu` on `WebContents`

The `inputFormType` property of the params object in the `context-menu`
event from `WebContents` has been removed. Use the new `formControlType`
property instead.

### Removed: `process.getIOCounters()`

Chromium has removed access to this information.

## Planned Breaking API Changes (29.0)

### Behavior Changed: `ipcRenderer` can no longer be sent over the `contextBridge`

Attempting to send the entire `ipcRenderer` module as an object over the `contextBridge` will now result in
an empty object on the receiving side of the bridge. This change was made to remove / mitigate
a security footgun. You should not directly expose ipcRenderer or its methods over the bridge.
Instead, provide a safe wrapper like below:

```js
contextBridge.exposeInMainWorld('app', {
  onEvent: (cb) => ipcRenderer.on('foo', (e, ...args) => cb(args))
})
```

### Removed: `renderer-process-crashed` event on `app`

The `renderer-process-crashed` event on `app` has been removed.
Use the new `render-process-gone` event instead.

```js
// Removed
app.on('renderer-process-crashed', (event, webContents, killed) => { /* ... */ })

// Replace with
app.on('render-process-gone', (event, webContents, details) => { /* ... */ })
```

### Removed: `crashed` event on `WebContents` and `<webview>`

The `crashed` events on `WebContents` and `<webview>` have been removed.
Use the new `render-process-gone` event instead.

```js
// Removed
win.webContents.on('crashed', (event, killed) => { /* ... */ })
webview.addEventListener('crashed', (event) => { /* ... */ })

// Replace with
win.webContents.on('render-process-gone', (event, details) => { /* ... */ })
webview.addEventListener('render-process-gone', (event) => { /* ... */ })
```

### Removed: `gpu-process-crashed` event on `app`

The `gpu-process-crashed` event on `app` has been removed.
Use the new `child-process-gone` event instead.

```js
// Removed
app.on('gpu-process-crashed', (event, killed) => { /* ... */ })

// Replace with
app.on('child-process-gone', (event, details) => { /* ... */ })
```

## Planned Breaking API Changes (28.0)

### Behavior Changed: `WebContents.backgroundThrottling` set to false affects all `WebContents` in the host `BrowserWindow`

`WebContents.backgroundThrottling` set to false will disable frames throttling
in the `BrowserWindow` for all `WebContents` displayed by it.

### Removed: `BrowserWindow.setTrafficLightPosition(position)`

`BrowserWindow.setTrafficLightPosition(position)` has been removed, the
`BrowserWindow.setWindowButtonPosition(position)` API should be used instead
which accepts `null` instead of `{ x: 0, y: 0 }` to reset the position to
system default.

```js
// Removed in Electron 28
win.setTrafficLightPosition({ x: 10, y: 10 })
win.setTrafficLightPosition({ x: 0, y: 0 })

// Replace with
win.setWindowButtonPosition({ x: 10, y: 10 })
win.setWindowButtonPosition(null)
```

### Removed: `BrowserWindow.getTrafficLightPosition()`

`BrowserWindow.getTrafficLightPosition()` has been removed, the
`BrowserWindow.getWindowButtonPosition()` API should be used instead
which returns `null` instead of `{ x: 0, y: 0 }` when there is no custom
position.

```js
// Removed in Electron 28
const pos = win.getTrafficLightPosition()
if (pos.x === 0 && pos.y === 0) {
  // No custom position.
}

// Replace with
const ret = win.getWindowButtonPosition()
if (ret === null) {
  // No custom position.
}
```

### Removed: `ipcRenderer.sendTo()`

The `ipcRenderer.sendTo()` API has been removed. It should be replaced by setting up a [`MessageChannebetween the renderers.

The `senderId` and `senderIsMainFrame` properties of `IpcRendererEvent` have been removed as well.

### Removed: `app.runningUnderRosettaTranslation`

The `app.runningUnderRosettaTranslation` property has been removed.
Use `app.runningUnderARM64Translation` instead.

```js
// Removed
console.log(app.runningUnderRosettaTranslation)
// Replace with
console.log(app.runningUnderARM64Translation)
```

### Deprecated: `renderer-process-crashed` event on `app`

The `renderer-process-crashed` event on `app` has been deprecated.
Use the new `render-process-gone` event instead.

```js
// Deprecated
app.on('renderer-process-crashed', (event, webContents, killed) => { /* ... */ })

// Replace with
app.on('render-process-gone', (event, webContents, details) => { /* ... */ })
```

### Deprecated: `params.inputFormType` property on `context-menu` on `WebContents`

The `inputFormType` property of the params object in the `context-menu`
event from `WebContents` has been deprecated. Use the new `formControlType`
property instead.

### Deprecated: `crashed` event on `WebContents` and `<webview>`

The `crashed` events on `WebContents` and `<webview>` have been deprecated.
Use the new `render-process-gone` event instead.

```js
// Deprecated
win.webContents.on('crashed', (event, killed) => { /* ... */ })
webview.addEventListener('crashed', (event) => { /* ... */ })

// Replace with
win.webContents.on('render-process-gone', (event, details) => { /* ... */ })
webview.addEventListener('render-process-gone', (event) => { /* ... */ })
```

### Deprecated: `gpu-process-crashed` event on `app`

The `gpu-process-crashed` event on `app` has been deprecated.
Use the new `child-process-gone` event instead.

```js
// Deprecated
app.on('gpu-process-crashed', (event, killed) => { /* ... */ })

// Replace with
app.on('child-process-gone', (event, details) => { /* ... */ })
```

## Planned Breaking API Changes (27.0)

### Removed: macOS 10.13 / 10.14 support

macOS 10.13 (High Sierra) and macOS 10.14 (Mojave) are no longer supported by 

Older versions of Electron will continue to run on these operating systems, but macOS 10.15 (Catalina)
or later will be required to run Electron v27.0.0 and higher.

### Deprecated: `ipcRenderer.sendTo()`

The `ipcRenderer.sendTo()` API has been deprecated. It should be replaced by setting up a [`MessageChannel between the renderers.

The `senderId` and `senderIsMainFrame` properties of `IpcRendererEvent` have been deprecated as well.

### Removed: color scheme events in `systemPreferences`

The following `systemPreferences` events have been removed:

* `inverted-color-scheme-changed`
* `high-contrast-color-scheme-changed`

Use the new `updated` event on the `nativeTheme` module instead.

```js
// Removed
systemPreferences.on('inverted-color-scheme-changed', () => { /* ... */ })
systemPreferences.on('high-contrast-color-scheme-changed', () => { /* ... */ })

// Replace with
nativeTheme.on('updated', () => { /* ... */ })
```

### Removed: Some `window.setVibrancy` options on macOS

The following vibrancy options have been removed:

* 'light'
* 'medium-light'
* 'dark'
* 'ultra-dark'
* 'appearance-based'

These were previously deprecated and have been removed by Apple in 10.15.

### Removed: `webContents.getPrinters`

The `webContents.getPrinters` method has been removed. Use
`webContents.getPrintersAsync` instead.

```js
const w = new BrowserWindow({ show: false })

// Removed
console.log(w.webContents.getPrinters())
// Replace with
w.webContents.getPrintersAsync().then((printers) => {
  console.log(printers)
})
```

### Removed: `systemPreferences.{get,set}AppLevelAppearance` and `systemPreferences.appLevelAppearance`

The `systemPreferences.getAppLevelAppearance` and `systemPreferences.setAppLevelAppearance`
methods have been removed, as well as the `systemPreferences.appLevelAppearance` property.
Use the `nativeTheme` module instead.

```js
// Removed
systemPreferences.getAppLevelAppearance()
// Replace with
nativeTheme.shouldUseDarkColors

// Removed
systemPreferences.appLevelAppearance
// Replace with
nativeTheme.shouldUseDarkColors

// Removed
systemPreferences.setAppLevelAppearance('dark')
// Replace with
nativeTheme.themeSource = 'dark'
```

### Removed: `alternate-selected-control-text` value for `systemPreferences.getColor`

The `alternate-selected-control-text` value for `systemPreferences.getColor`
has been removed. Use `selected-content-background` instead.

```js
// Removed
systemPreferences.getColor('alternate-selected-control-text')
// Replace with
systemPreferences.getColor('selected-content-background')
```

## Planned Breaking API Changes (26.0)

### Deprecated: `webContents.getPrinters`

The `webContents.getPrinters` method has been deprecated. Use
`webContents.getPrintersAsync` instead.

```js
const w = new BrowserWindow({ show: false })

// Deprecated
console.log(w.webContents.getPrinters())
// Replace with
w.webContents.getPrintersAsync().then((printers) => {
  console.log(printers)
})
```

### Deprecated: `systemPreferences.{get,set}AppLevelAppearance` and `systemPreferences.appLevelAppearance`

The `systemPreferences.getAppLevelAppearance` and `systemPreferences.setAppLevelAppearance`
methods have been deprecated, as well as the `systemPreferences.appLevelAppearance` property.
Use the `nativeTheme` module instead.

```js
// Deprecated
systemPreferences.getAppLevelAppearance()
// Replace with
nativeTheme.shouldUseDarkColors

// Deprecated
systemPreferences.appLevelAppearance
// Replace with
nativeTheme.shouldUseDarkColors

// Deprecated
systemPreferences.setAppLevelAppearance('dark')
// Replace with
nativeTheme.themeSource = 'dark'
```

### Deprecated: `alternate-selected-control-text` value for `systemPreferences.getColor`

The `alternate-selected-control-text` value for `systemPreferences.getColor`
has been deprecated. Use `selected-content-background` instead.

```js
// Deprecated
systemPreferences.getColor('alternate-selected-control-text')
// Replace with
systemPreferences.getColor('selected-content-background')
```

## Planned Breaking API Changes (25.0)

### Deprecated: `protocol.{un,}{register,intercept}{Buffer,String,Stream,File,Http}Protocol` and `protocol.isProtocol{Registered,Intercepted}`

The `protocol.register*Protocol` and `protocol.intercept*Protocol` methods have
been replaced with [`protocol.handle
The new method can either register a new protocol or intercept an existing
protocol, and responses can be of any type.

```js
// Deprecated in Electron 25
protocol.registerBufferProtocol('some-protocol', () => {
  callback({ mimeType: 'text/html', data: Buffer.from('<h5>Response</h5>') })
})

// Replace with
protocol.handle('some-protocol', () => {
  return new Response(
    Buffer.from('<h5>Response</h5>'), // Could also be a string or ReadableStream.
    { headers: { 'content-type': 'text/html' } }
  )
})
```

```js
// Deprecated in Electron 25
protocol.registerHttpProtocol('some-protocol', () => {
  callback({ url: 'https://electronjs.org' })
})

// Replace with
protocol.handle('some-protocol', () => {
  return net.fetch('https://electronjs.org')
})
```

```js
// Deprecated in Electron 25
protocol.registerFileProtocol('some-protocol', () => {
  callback({ filePath: '/path/to/my/file' })
})

// Replace with
protocol.handle('some-protocol', () => {
  return net.fetch('file:///path/to/my/file')
})
```

### Deprecated: `BrowserWindow.setTrafficLightPosition(position)`

`BrowserWindow.setTrafficLightPosition(position)` has been deprecated, the
`BrowserWindow.setWindowButtonPosition(position)` API should be used instead
which accepts `null` instead of `{ x: 0, y: 0 }` to reset the position to
system default.

```js
// Deprecated in Electron 25
win.setTrafficLightPosition({ x: 10, y: 10 })
win.setTrafficLightPosition({ x: 0, y: 0 })

// Replace with
win.setWindowButtonPosition({ x: 10, y: 10 })
win.setWindowButtonPosition(null)
```

### Deprecated: `BrowserWindow.getTrafficLightPosition()`

`BrowserWindow.getTrafficLightPosition()` has been deprecated, the
`BrowserWindow.getWindowButtonPosition()` API should be used instead
which returns `null` instead of `{ x: 0, y: 0 }` when there is no custom
position.

```js
// Deprecated in Electron 25
const pos = win.getTrafficLightPosition()
if (pos.x === 0 && pos.y === 0) {
  // No custom position.
}

// Replace with
const ret = win.getWindowButtonPosition()
if (ret === null) {
  // No custom position.
}
```

## Planned Breaking API Changes (24.0)

### API Changed: `nativeImage.createThumbnailFromPath(path, size)`

The `maxSize` parameter has been changed to `size` to reflect that the size passed in will be the size the thumbnail created. Previously, Windows would not scale the image up if it were smaller than `maxSize`, and
macOS would always set the size to `maxSize`. Behavior is now the same across platforms.

Updated Behavior:

```js
// a 128x128 image.
const imagePath = path.join('path', 'to', 'capybara.png')

// Scaling up a smaller image.
const upSize = { width: 256, height: 256 }
nativeImage.createThumbnailFromPath(imagePath, upSize).then(result => {
  console.log(result.getSize()) // { width: 256, height: 256 }
})

// Scaling down a larger image.
const downSize = { width: 64, height: 64 }
nativeImage.createThumbnailFromPath(imagePath, downSize).then(result => {
  console.log(result.getSize()) // { width: 64, height: 64 }
})
```

Previous Behavior (on Windows):

```js
// a 128x128 image
const imagePath = path.join('path', 'to', 'capybara.png')
const size = { width: 256, height: 256 }
nativeImage.createThumbnailFromPath(imagePath, size).then(result => {
  console.log(result.getSize()) // { width: 128, height: 128 }
})
```

## Planned Breaking API Changes (23.0)

### Behavior Changed: Draggable Regions on macOS

The implementation of draggable regions (using the CSS property `-webkit-app-region: drag`) has changed on macOS to bring it in line with Windows and Linux. Previously, when a region with `-webkit-app-region: no-drag` overlapped a region with `-webkit-app-region: drag`, the `no-drag` region would always take precedence on macOS, regardless of CSS layering. That is, if a `drag` region was above a `no-drag` region, it would be ignored. Beginning in Electron 23, a `drag` region on top of a `no-drag` region will correctly cause the region to be draggable.

Additionally, the `customButtonsOnHover` BrowserWindow property previously created a draggable region which ignored the `-webkit-app-region` CSS property. This has now been fixed (see [#37
As a result, if your app uses a frameless window with draggable regions on macOS, the regions which are draggable in your app may change in Electron 23.

### Removed: Windows 7 / 8 / 8.1 supp
Older versions of Electron will continue to run on these operating systems, but Windows 10 or later will be required to run Electron v23.0.0 and higher.

### Removed: BrowserWindow `scroll-touch-*` events

The deprecated `scroll-touch-begin`, `scroll-touch-end` and `scroll-touch-edge`
events on BrowserWindow have been removed. Instead, use the newly available
[`input-event` eventon WebContents.

```js
// Removed in Electron 23.0
win.on('scroll-touch-begin', scrollTouchBegin)
win.on('scroll-touch-edge', scrollTouchEdge)
win.on('scroll-touch-end', scrollTouchEnd)

// Replace with
win.webContents.on('input-event', (_, event) => {
  if (event.type === 'gestureScrollBegin') {
    scrollTouchBegin()
  } else if (event.type === 'gestureScrollUpdate') {
    scrollTouchEdge()
  } else if (event.type === 'gestureScrollEnd') {
    scrollTouchEnd()
  }
})
```

### Removed: `webContents.incrementCapturerCount(stayHidden, stayAwake)`

The `webContents.incrementCapturerCount(stayHidden, stayAwake)` function has been removed.
It is now automatically handled by `webContents.capturePage` when a page capture completes.

```js
const w = new BrowserWindow({ show: false })

// Removed in Electron 23
w.webContents.incrementCapturerCount()
w.capturePage().then(image => {
  console.log(image.toDataURL())
  w.webContents.decrementCapturerCount()
})

// Replace with
w.capturePage().then(image => {
  console.log(image.toDataURL())
})
```

### Removed: `webContents.decrementCapturerCount(stayHidden, stayAwake)`

The `webContents.decrementCapturerCount(stayHidden, stayAwake)` function has been removed.
It is now automatically handled by `webContents.capturePage` when a page capture completes.

```js
const w = new BrowserWindow({ show: false })

// Removed in Electron 23
w.webContents.incrementCapturerCount()
w.capturePage().then(image => {
  console.log(image.toDataURL())
  w.webContents.decrementCapturerCount()
})

// Replace with
w.capturePage().then(image => {
  console.log(image.toDataURL())
})
```

## Planned Breaking API Changes (22.0)

### Deprecated: `webContents.incrementCapturerCount(stayHidden, stayAwake)`

`webContents.incrementCapturerCount(stayHidden, stayAwake)` has been deprecated.
It is now automatically handled by `webContents.capturePage` when a page capture completes.

```js
const w = new BrowserWindow({ show: false })

// Removed in Electron 23
w.webContents.incrementCapturerCount()
w.capturePage().then(image => {
  console.log(image.toDataURL())
  w.webContents.decrementCapturerCount()
})

// Replace with
w.capturePage().then(image => {
  console.log(image.toDataURL())
})
```

### Deprecated: `webContents.decrementCapturerCount(stayHidden, stayAwake)`

`webContents.decrementCapturerCount(stayHidden, stayAwake)` has been deprecated.
It is now automatically handled by `webContents.capturePage` when a page capture completes.

```js
const w = new BrowserWindow({ show: false })

// Removed in Electron 23
w.webContents.incrementCapturerCount()
w.capturePage().then(image => {
  console.log(image.toDataURL())
  w.webContents.decrementCapturerCount()
})

// Replace with
w.capturePage().then(image => {
  console.log(image.toDataURL())
})
```

### Removed: WebContents `new-window` event

The `new-window` event of WebContents has been removed. It is replaced by [`webContents.setWindowOpenHandler(

```js
// Removed in Electron 22
webContents.on('new-window', (event) => {
  event.preventDefault()
})

// Replace with
webContents.setWindowOpenHandler((details) => {
  return { action: 'deny' }
})
```

### Removed: `<webview>` `new-window` event

The `new-window` event of `<webview>` has been removed. There is no direct replacement.

```js
// Removed in Electron 22
webview.addEventListener('new-window', (event) => {})
```

```js
// Replace with

// main.js
mainWindow.webContents.on('did-attach-webview', (event, wc) => {
  wc.setWindowOpenHandler((details) => {
    mainWindow.webContents.send('webview-new-window', wc.id, details)
    return { action: 'deny' }
  })
})

// preload.js
const { ipcRenderer } = require('electron')
ipcRenderer.on('webview-new-window', (e, webContentsId, details) => {
  console.log('webview-new-window', webContentsId, details)
  document.getElementById('webview').dispatchEvent(new Event('new-window'))
})

// renderer.js
document.getElementById('webview').addEventListener('new-window', () => {
  console.log('got new-window event')
})
```

### Deprecated: BrowserWindow `scroll-touch-*` events

The `scroll-touch-begin`, `scroll-touch-end` and `scroll-touch-edge` events on
BrowserWindow are deprecated. Instead, use the newly avail

```js
// Deprecated
win.on('scroll-touch-begin', scrollTouchBegin)
win.on('scroll-touch-edge', scrollTouchEdge)
win.on('scroll-touch-end', scrollTouchEnd)

// Replace with
win.webContents.on('input-event', (_, event) => {
  if (event.type === 'gestureScrollBegin') {
    scrollTouchBegin()
  } else if (event.type === 'gestureScrollUpdate') {
    scrollTouchEdge()
  } else if (event.type === 'gestureScrollEnd') {
    scrollTouchEnd()
  }
})
```

## Planned Breaking API Changes (21.0)

### Behavior Changed: V8 Memory Cage enabled

The V8 memory cage has been enabled, which has implications for native modules
which wrap non-V8 memory with `ArrayBuffer` or `Buffer`. See the
[blog post about the V8 memory cageor
more details.

### API Changed: `webContents.printToPDF()`

`webContents.printToPDF()` has been modified to conform to [`Page.printToP in the Chrome DevTools Protocol. This has been changes in order to
address changes upstream that made our previous implementation untenable and rife with bugs.

**Arguments Changed**

* `pageRanges`

**Arguments Removed**

* `printSelectionOnly`
* `marginsType`
* `headerFooter`
* `scaleFactor`

**Arguments Added**

* `headerTemplate`
* `footerTemplate`
* `displayHeaderFooter`
* `margins`
* `scale`
* `preferCSSPageSize`

```js
// Main process
const { webContents } = require('electron')

webContents.printToPDF({
  landscape: true,
  displayHeaderFooter: true,
  printBackground: true,
  scale: 2,
  pageSize: 'Ledger',
  margins: {
    top: 2,
    bottom: 2,
    left: 2,
    right: 2
  },
  pageRanges: '1-5, 8, 11-13',
  headerTemplate: '<h1>Title</h1>',
  footerTemplate: '<div><span class="pageNumber"></span></div>',
  preferCSSPageSize: true
}).then(data => {
  fs.writeFile(pdfPath, data, (error) => {
    if (error) throw error
    console.log(`Wrote PDF successfully to ${pdfPath}`)
  })
}).catch(error => {
  console.log(`Failed to write PDF to ${pdfPath}: `, error)
})
```

## Planned Breaking API Changes (20.0)

### Removed: macOS 10.11 / 10.12 support

macOS 10.11 (El Capitan) and macOS 10.12 (Sierra) are no longer supported by [Chromium

Older versions of Electron will continue to run on these operating systems, but macOS 10.13 (High Sierra)
or later will be required to run Electron v20.0.0 and higher.

### Default Changed: renderers without `nodeIntegration: true` are sandboxed by default

Previously, renderers that specified a preload script defaulted to being
unsandboxed. This meant that by default, preload scripts had access to Node.js.
In Electron 20, this default has changed. Beginning in Electron 20, renderers
will be sandboxed by default, unless `nodeIntegration: true` or `sandbox: false`
is specified.

If your preload scripts do not depend on Node, no action is needed. If your
preload scripts _do_ depend on Node, either refactor them to remove Node usage
from the renderer, or explicitly specify `sandbox: false` for the relevant
renderers.

### Removed: `skipTaskbar` on Linux

On X11, `skipTaskbar` sends a `_NET_WM_STATE_SKIP_TASKBAR` message to the X11
window manager. There is not a direct equivalent for Wayland, and the known
workarounds have unacceptable tradeoffs (e.g. Window.is_skip_taskbar in GNOME
requires unsafe mode), so Electron is unable to support this feature on Linux.

### API Changed: `session.setDevicePermissionHandler(handler)`

The handler invoked when `session.setDevicePermissionHandler(handler)` is used
has a change to its arguments.  This handler no longer is passed a frame
[`WebFrameMain`d the `origin`, which
is the origin that is checking for device permission.

## Planned Breaking API Changes (19.0)

### Removed: IA32 Linux binaries

This is a result of Chromium 102.0.4999.0 dropping support for IA32 Linux.
This concludes the [removal of support for IA32 Linux](#removed-ia32-linux-support).

## Planned Breaking API Changes (18.0)

### Removed: `nativeWindowOpen`

Prior to Electron 15, `window.open` was by default shimmed to use
`BrowserWindowProxy`. This meant that `window.open('about:blank')` did not work
to open synchronously scriptable child windows, among other incompatibilities.
Since Electron 15, `nativeWindowOpen` has been enabled by default.

See the documentation 
for more details.

## Planned Breaking API Changes (17.0)

### Removed: `desktopCapturer.getSources` in the renderer

The `desktopCapturer.getSources` API is now only available in the main process.
This has been changed in order to improve the default security of Electron
apps.

If you need this functionality, it can be replaced as follows:

```js
// Main process
const { ipcMain, desktopCapturer } = require('electron')

ipcMain.handle(
  'DESKTOP_CAPTURER_GET_SOURCES',
  (event, opts) => desktopCapturer.getSources(opts)
)
```

```js
// Renderer process
const { ipcRenderer } = require('electron')

const desktopCapturer = {
  getSources: (opts) => ipcRenderer.invoke('DESKTOP_CAPTURER_GET_SOURCES', opts)
}
```

However, you should consider further restricting the information returned to
the renderer; for instance, displaying a source selector to the user and only
returning the selected source.

### Deprecated: `nativeWindowOpen`

Prior to Electron 15, `window.open` was by default shimmed to use
`BrowserWindowProxy`. This meant that `window.open('about:blank')` did not work
to open synchronously scriptable child windows, among other incompatibilities.
Since Electron 15, `nativeWindowOpen` has been enabled by default.

See the documentation for [window.open in Ele
for more details.

## Planned Breaking API Changes (16.0)

### Behavior Changed: `crashReporter` implementation switched to Crashpad on Linux

The underlying implementation of the `crashReporter` API on Linux has changed
from Breakpad to Crashpad, bringing it in line with Windows and Mac. As a
result of this, child processes are now automatically monitored, and calling
`process.crashReporter.start` in Node child processes is no longer needed (and
is not advisable, as it will start a second instance of the Crashpad reporter).

There are also some subtle changes to how annotations will be reported on
Linux, including that long values will no longer be split between annotations
appended with `__1`, `__2` and so on, and instead will be truncated at the
(new, longer) annotation value limit.

### Deprecated: `desktopCapturer.getSources` in the renderer

Usage of the `desktopCapturer.getSources` API in the renderer has been
deprecated and will be removed. This change improves the default security of
Electron apps.

See [here for details on
how to replace this API in your app.

## Planned Breaking API Changes (15.0)

### Default Changed: `nativeWindowOpen` defaults to `true`

Prior to Electron 15, `window.open` was by default shimmed to use
`BrowserWindowProxy`. This meant that `window.open('about:blank')` did not work
to open synchronously scriptable child windows, among other incompatibilities.
`nativeWindowOpen` is no longer experimental, and is now the default.

See the documentation for [window.open in
for more details.

### Deprecated: `app.runningUnderRosettaTranslation`

The `app.runningUnderRosettaTranslation` property has been deprecated.
Use `app.runningUnderARM64Translation` instead.

```js
// Deprecated
console.log(app.runningUnderRosettaTranslation)
// Replace with
console.log(app.runningUnderARM64Translation)
```

## Planned Breaking API Changes (14.0)

### Removed: `remote` module

The `remote` module was deprecated in Electron 12, and will be removed in
Electron 14. It is replaced by the


```js
// Deprecated in Electron 12:
const { BrowserWindow } = require('electron').remote
```

```js
// Replace with:
const { BrowserWindow } = require('@electron/remote')

// In the main process:
require('@electron/remote/main').initialize()
```

### Removed: `app.allowRendererProcessReuse`

The `app.allowRendererProcessReuse` property will be removed as part of our plan to
more closely align with Chromium's process model for security, performance and maintainability.

For more detailed information see [#18397

### Removed: Browser Window Affinity

The `affinity` option when constructing a new `BrowserWindow` will be removed
as part of our plan to more closely align with Chromium's process model for security,
performance and maintainability.

For more detailed information see [#1839

### API Changed: `window.open()`

The optional parameter `frameName` will no longer set the title of the window. This now follows the specification described by the [native under the corresponding parameter `windowName`.

If you were using this parameter to set the title of a window, you can instead use 

### Removed: `worldSafeExecuteJavaScript`

In Electron 14, `worldSafeExecuteJavaScript` will be removed.  There is no alternative, please
ensure your code works with this property enabled.  It has been enabled by default since Electron
12.

You will be affected by this change if you use either `webFrame.executeJavaScript` or `webFrame.executeJavaScriptInIsolatedWorld`. You will need to ensure that values returned by either of those methods are supported by the [Context Bridge API](api/context-as these methods use the same value passing semantics.

### Removed: BrowserWindowConstructorOptions inheriting from parent windows

Prior to Electron 14, windows opened with `window.open` would inherit
BrowserWindow constructor options such as `transparent` and `resizable` from
their parent window. Beginning with Electron 14, this behavior is removed, and
windows will not inherit any BrowserWindow constructor options from their
parents.

Instead, explicitly set options for the new window with `setWindowOpenHandler`:

```js
webContents.setWindowOpenHandler((details) => {
  return {
    action: 'allow',
    overrideBrowserWindowOptions: {
      // ...
    }
  }
})
```

### Removed: `additionalFeatures`

The deprecated `additionalFeatures` property in the `new-window` and
`did-create-window` events of WebContents has been removed. Since `new-window`
uses positional arguments, the argument is still present, but will always be
the empty array `[]`. (Though note, the `new-window` event itself is
deprecated, and is replaced by `setWindowOpenHandler`.) Bare keys in window
features will now present as keys with the value `true` in the options object.

```js
// Removed in Electron 14
// Triggered by window.open('...', '', 'my-key')
webContents.on('did-create-window', (window, details) => {
  if (details.additionalFeatures.includes('my-key')) {
    // ...
  }
})

// Replace with
webContents.on('did-create-window', (window, details) => {
  if (details.options['my-key']) {
    // ...
  }
})
```

## Planned Breaking API Changes (13.0)

### API Changed: `session.setPermissionCheckHandler(handler)`

The `handler` methods first parameter was previously always a `webContents`, it can now sometimes be `null`.  You should use the `requestingOrigin`, `embeddingOrigin` and `securityOrigin` properties to respond to the permission check correctly.  As the `webContents` can be `null` it can no longer be relied on.

```js
// Old code
session.setPermissionCheckHandler((webContents, permission) => {
  if (webContents.getURL().startsWith('https://google.com/') && permission === 'notification') {
    return true
  }
  return false
})

// Replace with
session.setPermissionCheckHandler((webContents, permission, requestingOrigin) => {
  if (new URL(requestingOrigin).hostname === 'google.com' && permission === 'notification') {
    return true
  }
  return false
})
```

### Removed: `shell.moveItemToTrash()`

The deprecated synchronous `shell.moveItemToTrash()` API has been removed. Use
the asynchronous `shell.trashItem()` instead.

```js
// Removed in Electron 13
shell.moveItemToTrash(path)
// Replace with
shell.trashItem(path).then(/* ... */)
```

### Removed: `BrowserWindow` extension APIs

The deprecated extension APIs have been removed:

* `BrowserWindow.addExtension(path)`
* `BrowserWindow.addDevToolsExtension(path)`
* `BrowserWindow.removeExtension(name)`
* `BrowserWindow.removeDevToolsExtension(name)`
* `BrowserWindow.getExtensions()`
* `BrowserWindow.getDevToolsExtensions()`

Use the session APIs instead:

* `ses.loadExtension(path)`
* `ses.removeExtension(extension_id)`
* `ses.getAllExtensions()`

```js
// Removed in Electron 13
BrowserWindow.addExtension(path)
BrowserWindow.addDevToolsExtension(path)
// Replace with
session.defaultSession.loadExtension(path)
```

```js
// Removed in Electron 13
BrowserWindow.removeExtension(name)
BrowserWindow.removeDevToolsExtension(name)
// Replace with
session.defaultSession.removeExtension(extension_id)
```

```js
// Removed in Electron 13
BrowserWindow.getExtensions()
BrowserWindow.getDevToolsExtensions()
// Replace with
session.defaultSession.getAllExtensions()
```

### Removed: methods in `systemPreferences`

The following `systemPreferences` methods have been deprecated:

* `systemPreferences.isDarkMode()`
* `systemPreferences.isInvertedColorScheme()`
* `systemPreferences.isHighContrastColorScheme()`

Use the following `nativeTheme` properties instead:

* `nativeTheme.shouldUseDarkColors`
* `nativeTheme.shouldUseInvertedColorScheme`
* `nativeTheme.shouldUseHighContrastColors`

```js
// Removed in Electron 13
systemPreferences.isDarkMode()
// Replace with
nativeTheme.shouldUseDarkColors

// Removed in Electron 13
systemPreferences.isInvertedColorScheme()
// Replace with
nativeTheme.shouldUseInvertedColorScheme

// Removed in Electron 13
systemPreferences.isHighContrastColorScheme()
// Replace with
nativeTheme.shouldUseHighContrastColors
```

### Deprecated: WebContents `new-window` event

The `new-window` event of WebContents has been deprecated. It is replaced by [`webContents.setWindowOpenHandler()`

```js
// Deprecated in Electron 13
webContents.on('new-window', (event) => {
  event.preventDefault()
})

// Replace with
webContents.setWindowOpenHandler((details) => {
  return { action: 'deny' }
})
```

## Planned Breaking API Changes (12.0)

### Removed: Pepper Flash support

Chromium has removed support for Flash, and so we must follow suit. See
Chromium's  for more
details.

### Default Changed: `worldSafeExecuteJavaScript` defaults to `true`

In Electron 12, `worldSafeExecuteJavaScript` will be enabled by default.  To restore
the previous behavior, `worldSafeExecuteJavaScript: false` must be specified in WebPreferences.
Please note that setting this option to `false` is **insecure**.

This option will be removed in Electron 14 so please migrate your code to support the default
value.

### Default Changed: `contextIsolation` defaults to `true`

In Electron 12, `contextIsolation` will be enabled by default.  To restore
the previous behavior, `contextIsolation: false` must be specified in WebPreferences.

Wefor the security of your application.

Another implication is that `require()` cannot be used in the renderer process unless
`nodeIntegration` is `true` and `contextIsolation` is `false`.

For more details see: https://github.com/electron/electron/issues/23506

### Removed: `crashReporter.getCrashesDirectory()`

The `crashReporter.getCrashesDirectory` method has been removed. Usage
should be replaced by `app.getPath('crashDumps')`.

```js
// Removed in Electron 12
crashReporter.getCrashesDirectory()
// Replace with
app.getPath('crashDumps')
```

### Removed: `crashReporter` methods in the renderer process

The following `crashReporter` methods are no longer available in the renderer
process:

* `crashReporter.start`
* `crashReporter.getLastCrashReport`
* `crashReporter.getUploadedReports`
* `crashReporter.getUploadToServer`
* `crashReporter.setUploadToServer`
* `crashReporter.getCrashesDirectory`

They should be called only from the main process.

See [#2326 for more details.

### Default Changed: `crashReporter.start({ compress: true })`

The default value of the `compress` option to `crashReporter.start` has changed
from `false` to `true`. This means that crash dumps will be uploaded to the
crash ingestion server with the `Content-Encoding: gzip` header, and the body
will be compressed.

If your crash ingestion server does not support compressed payloads, you can
turn off compression by specifying `{ compress: false }` in the crash reporter
options.

### Deprecated: `remote` module

The `remote` module is deprecated in Electron 12, and will be removed in
Electron 14. It is replaced by the.

```js
// Deprecated in Electron 12:
const { BrowserWindow } = require('electron').remote
```

```js
// Replace with:
const { BrowserWindow } = require('@electron/remote')

// In the main process:
require('@electron/remote/main').initialize()
```

### Deprecated: `shell.moveItemToTrash()`

The synchronous `shell.moveItemToTrash()` has been replaced by the new,
asynchronous `shell.trashItem()`.

```js
// Deprecated in Electron 12
shell.moveItemToTrash(path)
// Replace with
shell.trashItem(path).then(/* ... */)
```

## Planned Breaking API Changes (11.0)

### Removed: `BrowserView.{destroy, fromId, fromWebContents, getAllViews}` and `id` property of `BrowserView`

The experimental APIs `BrowserView.{destroy, fromId, fromWebContents, getAllViews}`
have now been removed. Additionally, the `id` property of `BrowserView`
has also been removed.

For more detailed information, see [#23578](htt

## Planned Breaking API Changes (10.0)

### Deprecated: `companyName` argument to `crashReporter.start()`

The `companyName` argument to `crashReporter.start()`, which was previously
required, is now optional, and further, is deprecated. To get the same
behavior in a non-deprecated way, you can pass a `companyName` value in
`globalExtra`.

```js
// Deprecated in Electron 10
crashReporter.start({ companyName: 'Umbrella Corporation' })
// Replace with
crashReporter.start({ globalExtra: { _companyName: 'Umbrella Corporation' } })
```

### Deprecated: `crashReporter.getCrashesDirectory()`

The `crashReporter.getCrashesDirectory` method has been deprecated. Usage
should be replaced by `app.getPath('crashDumps')`.

```js
// Deprecated in Electron 10
crashReporter.getCrashesDirectory()
// Replace with
app.getPath('crashDumps')
```

### Deprecated: `crashReporter` methods in the renderer process

Calling the following `crashReporter` methods from the renderer process is
deprecated:

* `crashReporter.start`
* `crashReporter.getLastCrashReport`
* `crashReporter.getUploadedReports`
* `crashReporter.getUploadToServer`
* `crashReporter.setUploadToServer`
* `crashReporter.getCrashesDirectory`

The only non-deprecated methods remaining in the `crashReporter` module in the
renderer are `addExtraParameter`, `removeExtraParameter` and `getParameters`.

All above methods remain non-deprecated when called from the main process) for more details.

### Deprecated: `crashReporter.start({ compress: false })`

Setting `{ compress: false }` in `crashReporter.start` is deprecated. Nearly
all crash ingestion servers support gzip compression. This option will be
removed in a future version of Electron.

### Default Changed: `enableRemoteModule` defaults to `false`

In Electron 9, using the remote module without expl
