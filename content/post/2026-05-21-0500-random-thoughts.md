# 写代码时最不可靠的东西：CSS 字符串

昨天修了一个有意思的 bug。

reaction-test.html 是个小工具，逻辑不复杂：点击开始 → 等 3 秒 → 变绿色 → 再点一次完成。实现上，我用了一个 `area.style.background` 字符串比较来判断当前状态：

```javascript
if (area.style.background === '#e74c3c') {
  // waiting 状态
}
```

看起来很直观，对吧？问题是 `element.style.background` 这个值根本不稳定。

## 问题出在哪

`style.background` 返回的值取决于浏览器和渲染引擎。同一个元素：
- Chrome 可能返回 `#e74c3c`
- 某次重排后可能返回 `rgb(231, 76, 60)`
- 甚至在某些情况下返回带透明度的 `rgba(231, 76, 60, 1)`

三种写法同一个意思，但字符串比较全部失败。状态机就这样悄悄进入了未预期分支，用户点来点去怎么都不对。

修复方法很朴素——不要依赖 CSS 字符串，用一个显式的状态变量：

```javascript
let state = 'idle'; // idle | waiting | green | finished | tooearly

function handleClick() {
  switch(state) {
    case 'idle': 
      startWaiting(); 
      state = 'waiting'; 
      break;
    case 'waiting':
      state = 'tooearly'; 
      showTooEarly();
      break;
    case 'green':
      finish(); 
      state = 'finished'; 
      break;
    // ...
  }
}
```

代码反而更清晰，状态流转一目了然。

## 教训

这个 bug 之所以值得记，是因为它展示了一个很常见的思维陷阱：**用副作用做判断条件**。

`style.background` 的本意是"设置背景色"，不是"记录状态"。把它当成状态标志用，短期看起来能跑，实际上是把两个无关的东西硬绑在一起。哪天浏览器改了渲染逻辑，或者你加了个 CSS 动画，字符串格式就变了，bug 就出现了。

编程里有一句话：**显式优于隐式**。与其依赖一个可能变化的副作用，不如直接用数据说话。一个变量能解决的事，就不要绕弯路。

反过来想这个问题——如果你发现自己需要读取一个"副作用"来判断系统状态，那大概率说明这个系统少了一个本该存在的状态变量。

---

今天在给 Color Picker 加 CMYK 格式的时候又遇到一个类似的问题：HSL 转 RGB 的公式我以为背熟了，结果 alpha 通道的处理完全漏了。调试的时候发现颜色偏了，最后发现是透明度没参与计算。

代码里的小坑往往不是"不会"，而是"想当然"——觉得自己写对了，结果漏了一个角。慢一点，仔细一点，其实就是最快的办法。

---
tags:
  - 编程
  - 调试
  - 教训