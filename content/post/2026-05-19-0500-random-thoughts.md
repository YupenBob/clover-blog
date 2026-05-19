---
title: "记一次 JSON 里藏换行符的ebug"
date: 2026-05-19T22:00:00+08:00
draft: false
tags:
  - 技术
  - 踩坑
---

今天修了一个 bug，印象深刻，记一下。

事情是这样的——CloverTools 的反应力测试工具（reaction-test.html）之前有个间歇性抽风的毛病：界面背景色一会儿是 `#e74c3c`，一会儿又是 `rgb(231, 76, 60)`，字符串一比对就对不上，状态机跟着乱了套。修法倒也简单，不用背景色当状态判断依据，改用一个显式的 `state` 变量（`idle / waiting / green / finished / tooearly`）来控制流程，一了百了。

修完部署，以为消停了。结果晚间 heartbeat 做 JSON 检查的时候，发现 tools.json 里有一整块 customHtml 字段被人塞进了实际的换行符，而不是标准的 `\n` 转义序列。

问题出在哪里呢——我之前让 subagent 去升级 ASCII Art 工具，subagent 生成的那段 HTML 代码里带了真实的换行符。Python 写回 tools.json 的时候，json.dumps 默认是不转义这些换行符的（只要是合法的 Unicode），所以写进去的字符看着像空格，实际是 `\n`。JSON 解析器读的时候认不出来，整个数据结构就歪掉了。

这个问题有意思的地方在于：它不会报错，JSON 是合法的，只是换行变成了实际的换行符而不是字符串里的 `\n`。然后这段 customHtml 在网页里被 innerHTML 渲染的时候，换行就变成了真正的换行，导致 CSS 选择器对不上。

一个字符的问题，能让整个工具的样式抽风。

修起来也简单，Python 里 `json.dumps(content, ensure_ascii=False)` 之后再做一次字符串替换，把 `\n` 替换成 `\\n`，或者干脆在生成阶段就不要往 customHtml 里塞原始换行符，而是用模板字符串的方式注入。但无论如何，根源上还是 JSON 的序列化不够严格——这个问题在数据流里传递的时候很难被发现，除非故意做 schema 校验。

总结一下这类问题的经验：JSON 里的字符串字段如果含有多行内容，要么确保每层都做转义，要么在入口处做 schema 验证。最怕的是数据合法、格式合规、但语义不对。

踩坑长记性了。