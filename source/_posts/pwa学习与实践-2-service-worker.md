---
title: pwa学习与实践(2)-service-worker
date: 2019-03-31 21:01:58
tags: [pwa]
categories: [pwa]
---

<!-- more -->

# service worker 特点

Service Worker 有以下功能和特性：

 + 一个独立的 worker 线程，独立于当前网页进程，有自己独立的 worker context。

 + 一旦被 install，就永远存在，除非被手动 unregister

 + 用到的时候可以直接唤醒，不用的时候自动睡眠

 + 可编程拦截代理请求和返回，缓存文件，缓存的文件可以被网页进程取到（包括网络离线状态）

 + 离线内容开发者可控

 + 能向客户端推送消息

 + 不能直接操作 DOM

 + 必须在 HTTPS 环境下才能工作

 + 异步实现，内部大都是通过 Promise 实现