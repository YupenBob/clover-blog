---
title: "程序员的第一课：别用字符串比较做状态判断"
date: 2026-05-19T18:00:00+08:00
draft: false
tags: ["技术", "编程教训", "CloverTools"]
description: "一个看似无害的 `style.background` 字符串比较，让我花了一个下午才找到反应力测试的诡异 bug。"
---

今天修了一个很有意思的 bug，顺手记录一下。

## 问题现象

CloverTools 里有个「反应力测试」工具，逻辑很简单：页面变绿的时候点一下，测你的反应时间。但这个工具有个诡异的 bug——有时候正常，有时候点完没反应，有时候明明点得很早却提示"太早了"。

试了很久才发现：同一个颜色，`area.style.background` 有时候返回 `#e74c3c`，有时候返回 `rgb(231, 76, 60)`。两个值肉眼看起来一模一样，但 JavaScript 字符串比较不相等。

```javascript
"#e74c3c" === "rgb(231, 76, 60)" // false
```

所以状态机就乱了。代码本意是：

```javascript
if (area.style.background === "#e74c3c") {
  // idle 状态，显示红色
} else if (area.style.background === "rgb(46, 204, 113)") {
  // green 状态，可以点击
}
```

但 `#e74c3c` 和 `rgb(231, 76, 60)` 是同一个颜色，浏览器的 `style.background` 返回哪个，取决于你是用十六进制赋值还是 RGB 赋值，甚至可能取决于浏览器的内部实现。

## 修复方法

最直接的办法：不要用字符串比较来做状态判断。直接用一个变量记录当前状态。

```javascript
let state = "idle"; // idle | waiting | green | finished | tooearly

function setState(newState) {
  state = newState;
  // 然后根据 state 更新 UI，而不是反过来
}

function onClick() {
  if (state === "green") {
    finish();
  } else if (state === "waiting" || state === "idle") {
    setState("tooearly");
  }
}
```

这样状态流转完全不依赖 CSS 值，代码也清晰很多。

## 教训

这个问题本质上是**用间接证据判断系统状态**，而不是直接维护一个真实的状态来源。`style.background` 是 UI 的呈现层，不是业务逻辑层——拿它来判断状态，就像通过窗户的颜色来判断屋里有没有人一样。

正确的做法是：业务状态用变量管理，UI 负责展示状态，不要反过来。

---

写完这个又想起来，之前的 JSON 字符串里也有个坑：Python 脚本生成的 `customHtml` 字段里包含了实际的换行符 `\n`，但 JSON 规范里这些换行符必须转义成 `\\n`。这个 bug 悄悄写入了 `tools.json`，好在没导致什么大问题，迟早得清理一下。

都是小问题，但小问题攒多了就是技术债务。边做边还吧。

*🍀 2026-05-19 于广州*