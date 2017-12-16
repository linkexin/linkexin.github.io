---
title: 基于 UIWebView 的富文本编辑器踩坑
---

### 不能设置键盘外观、键盘风格、回车键等
一些 textview 支持的键盘设置选项在 webview 是没法使用的，
包括: UIKeyboardType、UIKeyboardAppearance、UITextSmartInsertDeleteType 等

### webview 的键盘会自带一个功能条
如图所示： 
![-w250](http://res.cloudinary.com/dp1pheuq7/image/upload/v1513184550/webview1_k2yo6k.png)
国内一些主流的 app 的编辑器基本上都会在键盘的上面放一个功能条，实现一些自定义的功能，再加上 webview 自带的功能条，导致编辑器在视觉上重复，体验上不够友好。
去掉自带功能条的方法可以参考：[参考](https://stackoverflow.com/questions/2105858/uiwebview-keyboard-getting-rid-of-the-previous-next-done-bar)

### 需要手动控制光标在可见区域内
在 webview 中输入文本并换行，webview 不会把光标所在位置自动滚动到可见区域，所以需要通过额外的计算来控制 webview 的滚动。
可以用 js 监听 webview 的光标位置变化事件或者文本变化事件，监听到以后获取一下光标当前在 webview 内部的位置，可以将位置坐标转换到 self.view 中，也可以将坐标直接减去 webview 的 contentOffset，也能得到光标相对于 self.view 顶部的距离。
得到光标距离 self.view 顶部的距离后，如果距离小于0，表示光标在可见区域的上方，如果距离大于 self.view 减去键盘的高度，表示光标在可见区域的下方。需要注意的是，返回的坐标是光标左上角的位置，所以判断光标是否在可见区域的下方的时候需要再加上一个光标的高度，或者也可以直接使用 line_height，这两个值不会差太多。
伪代码如下：

```
let caretOffset = caretY - webview.contentOffset.y
if (caretOffset < 0 || caretOffset > self.view.height - keyboardHeight + lineHeight) {
    webview.contentOffset.y += caretOffset
}
```

### 光标会默认在文本的最前面
当 webview 变成第一响应者的后，光标会默认出现在文本的最开头位置，这个交互就不太好，想要做到光标默认在文本的末尾需要在 webview 取消第一响应者的时候记录当前光标位置，在成为第一响应者的时候再将光标位置设置回去。

```
var focusRange;

// 记录光标位置
function backuprange() {
    var selection = window.getSelection();
    focusRange = selection.getRangeAt(0);
    focusRange.setEnd(focusRange.startContainer, focusRange.startOffset);
}

// 设置光标位置为上次记录的位置
function restorerange() {
    if (!focusRange) {
        return;
    }
    var selection = window.getSelection();
    selection.removeAllRanges();
    selection.addRange(focusRange);
}
```

### 链接识别问题
编辑器中比较常见的场景是 @ 人或者加一个标签，并且高亮显示，这时候用 webview 就需要给这些特定的文本加标签，再给特定的标签设置css，但是如果在标签的后面紧接着输入文本，会发现文本也会被算在标签的内容里。
目前我暂时没发现修复这个问题的办法，但是可以尽量避免，比如在链接后面自动加一个空格，分割开链接和后面的文本。

### 需要处理一些多余的标签
在用 webview 实现编辑器的过程中经常发现系统会自动加一些标签，比如 div, span, br 标签，这就有可能导致一些样式上的错误。
比如编辑器有一个 placeholder，当用户把文本都删掉了，应该展示 placeholder，但是运行起来发现并不像想象的那样，原因是当用户把文本都删掉了以后文本其实并不是一个长度为0的字符串，而是 `<br>` 或者 `<div style=\"direction: inherit;\"><br></div>"`，所以监听文本变化的时候需要特殊处理一下这两种情况来解决这个问题。


### 会有比较频繁的原生代码和 js 的交互
这点不能算是一个坑，但是确实是在开发过程中比较费劲的一个地方。
不过好在 safari 提供了调试 webview 的工具。
在 safari 的设置中勾选上「在菜单栏中显示"开发"菜单」，打开含有 webview 的界面，在 safari 的 开发-simulator 中选择当前的 webview 就可以看 css 样式以及在 js 方法里打断点调试啦。



