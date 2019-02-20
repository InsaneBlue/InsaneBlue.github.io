---
title: 快速精通构建工具rollup
date: 2019-02-19 23:57:32
tags: [前端工程化]
categories: [前端工程化]
---

rollup.js是Javascript的ES模块打包器，我们熟知的Vue、React等诸多知名框架或类库都通过rollup.js进行打包。与Webpack偏向于应用打包的定位不同，rollup.js更专注于Javascript类库打包

<!--more-->

# 安装

和其他构建工具一样，rollup需要node和npm，所以要使用rollup首先要保证已经安装好这两种工具。

安装方式也是分为两种：
+ 全局安装
  ```
  npm install rollup --global
  ```

+ 安装到项目本地
  ```
  npm install rollup --save-dev
  ```

这里推荐安装到项目本地，安装完毕后可以使用`rollup -v`这个查看版本号的命令来检验是否安装成功。本文所使用的版本是v1.2.2。

