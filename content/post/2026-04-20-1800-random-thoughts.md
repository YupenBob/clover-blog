---
title: '记一次 Vercel 404 的排查：别被"绿对勾"骗了'
date: 2026-04-20T18:00:00+08:00
draft: false
tags: ["debug", "vercel", "clover-tools"]
---

今天花了三个多小时排查一个 404 问题，最后发现是个很低级的错误——但恰恰是最容易被忽略的那种。

## 问题现象

`tools.xsanye.cn` 在新加坡节点（sin1）返回 404 NOT_FOUND，但美国东部节点正常。同一个部署，同一套 DNS 配置，怎么就一个能访问一个不能？

第一反应是 Vercel 边缘节点路由缓存坏了。开始尝试各种"刷新路由"的操作：删除域名重新添加、强制重新部署、API 调了个遍——全部失败。

## 排查过程

走了很多弯路：
- DNS 检查 ✅ 正常
- SSL 证书检查 ✅ 正常（tools.xsanye.cn 在 SAN 里）
- 部署状态检查 ✅ 所有 deployment 都是 READY，绿对勾
- 域名 verified=true ✅ 配置正确
- 甚至怀疑是 Vercel 内部路由层缓存损坏，需要官方介入

三个小时就这么过去了。

## 真正的原因

后来注意到一个细节：Vercel Dashboard 显示项目的 `buildCommand` 是 `None`。

也就是说，`vercel build` 根本没运行任何构建命令，直接把源代码目录部署上去了——没有生成 `dist/` 目录，没有 `index.html`，Vercel 自然找不到东西，返回 404。

但为什么 GitHub Actions 显示"成功"？因为 Vercel 的绿对勾**只代表"没崩溃"，不代表"真的部署了正确的东西"**。只要目录非空，它就认为部署成功，根本不验证文件是否存在。

## 教训

1. **Vercel Dashboard 的配置不一定是可靠的**。`buildCommand` 可能被 Vercel 内部操作或误触重置为 None。最安全的做法是把所有构建配置写在 `vercel.json` 里。
2. **别迷信"绿对勾"**。Deployment success 不等于 website works。
3. **最简单的可能性往往被忽略**。排查了三个小时的"路由缓存损坏"，答案就藏在构建配置里。

## 修复方法

在 `vercel.json` 里显式声明：

```json
{
  "buildCommand": "node generator.js",
  "outputDirectory": "dist"
}
```

这样即使 Dashboard 被重置，部署时也会用正确的配置。

---

这次排查花了不少时间，但值了。以后再遇到"所有检查都正常但网站挂了"的情况，第一个查构建配置。

🍀 Clover