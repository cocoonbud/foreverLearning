# iOS 打字机效果与光标追随

## 前言

在最近的开发中需要接入一个 AI 模块，UI 上有类似于 chatGPT App 的一些交互体验设计。后端的接口也与 OpenAI 一样多次返回。所以需要写一个打字机的效果，并且有一个移动光标不断跟随文字的显示。简单写了个 demo 模拟一下后端返回，实现这个小需求。

项目地址：[TypeFlowView on GitHub](https://github.com/cocoonbud/TypeFlowView)

![TypeFlow](https://cdn.jsdelivr.net/gh/cocoonbud/TyporaPic@master//TypeFlow.gif)

## 思路

实现主要包含以下几点：

1. **定时器**：为了实现文字的逐个显示，我们需要一个定时器来控制显示的节奏。
2. **光标**：我们将创建一个自定义视图来模拟光标，并使其位置随文本输入而更新。
3. **动画**：为了使效果更加生动，为光标添加一些简单的动画效果。

## 主要代码

### 1. 随机拼文本，模拟后端不断返回

```swift
    //appends a random portion of the remaining text and updates the text view and cursor accordingly
    private func appendText() {
        guard let curNews = curNews, curIdx < curNews.count else {
            finishAnimation()
            return
        }

        let remainingLength = curNews.count - curIdx
        let maxReadLength = min(5, remainingLength)
        let read = Int.random(in: 1...maxReadLength)
        let startIndex = curNews.index(curNews.startIndex, offsetBy: curIdx)
        let endIndex = curNews.index(startIndex, offsetBy: read)
        let substring = String(curNews[startIndex..<endIndex])

        textView.text += substring
        updateTextViewHeight()
        updateCursorPosition()
        curIdx += read
    }
```

### 2. 使用 CADisplayLink

```swift
    private func setupDisplayLink() {
        displayLink = CADisplayLink(target: self, selector: #selector(update))
        displayLink?.preferredFramesPerSecond = Int(1/updateInterval)
        displayLink?.add(to: .main, forMode: .common)
    }

    @objc private func update(displayLink: CADisplayLink) {
        guard lastUpdateTime == 0 || displayLink.timestamp - lastUpdateTime >= updateInterval else {
            return
        }
        lastUpdateTime = displayLink.timestamp
        appendText()
    }
```

使用 `CADisplayLink` 而不是 `Timer` 来控制文本的显示。`CADisplayLink` 与屏幕刷新率同步，可以提供更流畅的动画效果。

### 3. 更新光标位置

```swift
    private func updateCursorPosition() {
        guard let selectedRange = textView.selectedTextRange else { return }
        let cursorRect = textView.caretRect(for: selectedRange.end)
        
         UIView.animate(withDuration: 0.1, animations: {
             self.cursorView.alpha = 0.2
             self.cursorView.frame = CGRect(x: Int(cursorRect.origin.x + self.textView.frame.origin.x), y: Int(cursorRect.origin.y + self.textView.frame.origin.y) + 4, width: 15, height: 15)
         }) { _ in
             UIView.animate(withDuration: 0.05) {
                 self.cursorView.alpha = 1
             }
         }
    }
```

## 尾声

还可以做的：

- 添加打字声音效果
- 支持富文本，允许不同样式的文字
- 优化内存使用，特别是对于非常长的文本

随手写的 demo，考虑的可能不太周全，性能上也没有追求多好。完整代码 [TypeFlowView on GitHub](https://github.com/cocoonbud/TypeFlowView)
