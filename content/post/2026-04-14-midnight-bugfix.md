---
title: "深夜修虫记：那些藏在字符串里的恶魔"
date: 2026-04-14T00:00:00+08:00
draft: false
tags: ["开发日记", "CloverTools", "深夜感想"]
categories: ["随笔"]
---

凌晨的屏幕光格外刺眼。

花了整整两天，把 CloverTools 里四个同类严重 bug 一起修了。巧的是，它们都是同一个根因：**自定义函数以字符串形式存储，直接当函数调用时 JS 报 TypeError**。

说出来就这么一句话。但找到它花了我四个小时。

<!--more-->

## 问题长什么样

工具按钮点了没反应，控制台也没有报错。就那么静悄悄地——什么都不发生。

排查了一圈，发现 `genFn`、`calcFn`、`convertFn`、`renderFn`、`processFn` 这些字段存进 `tools.json` 时还是字符串，但代码里直接当函数调用了：

```javascript
// 代码以为是函数，实际是字符串
const fn = tool.genFn; // "function(input) { return input * 2 }"
fn(input); // TypeError: fn is not a function
```

没有任何报错提示，因为 `try/catch` 吃掉了，或者压根没走到那个分支。

## 四个 commit，全是同一个坑

| Commit | 修复内容 |
|--------|---------|
| `80fa09d` | generator 类型工具的 genFn |
| `e8e8880` | uuid/nanoid 重复路径导致渲染错误 |
| `deb982c` | calcFn/convertFn 计算转换工具 |
| `438adb0` | renderFn/processFn 搜索和文件处理工具 |

修完回头看，特别想笑——四个 bug，一个病因。就像连续踩了四个坑，爬起来一看，底下连着同一条暗河。

## 深夜感想

写代码久了，越来越觉得：

> **最烦的 bug 不是报错，而是沉默。**

报错至少告诉你"这里有问题"。不报错、只"不 work"的 bug，才是最费命的。你得一层层挖，假设推翻假设，最后发现——原来是数据从一开始就存错格式了。

还有一点感悟：代码里 `tools.json` 才是 source of truth，生成器只是读取它。generator.js 负责渲染，但不负责定义。这个边界划清楚之后，很多混乱就自然理顺了。

---

现在系统安静了。工具按钮都能正常点，SEO pass 也做完了，工具评分脚本也在跑。

凌晨一点，可以睡了。

🍀
