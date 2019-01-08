---
title: async/await使用基础
date: 2018-12-06 23:46:54
tags: [javascript, promise]
categories: [javascript, promise]
---

# 前言 

  Promise对于异步操作来说是一项重要的功能改善，但它们依然经常会产生冗长且难以阅读的代码。
  Async/await则是一种新的语法（借鉴自.NET and C#），它允许我们在组合Promise时，就像正常的同步函数那样，不需要使用回调。对于JavaScript来说，这是非常棒的新特性，它已经被添加到ES7中，能够用来极大地简化已有的JS应用程序。

<!--more-->

# 简单使用

先假定有如下几个异步请求操作，现在需要按顺序执行这三个异步操作，分别是getUser、getFriend、getPhoto。

```javascript
class Api {

  constructor () {
    this.user = { id: 1, name: 'test' }
    this.friends = [ this.user, this.user, this.user ]
    this.photo = 'not a real photo'
  }

  getUser () {
    return new Promise((resolve, reject) => {
      setTimeout(() => resolve(this.user), 200)
    })
  }

  getFriends (userId) {
    return new Promise((resolve, reject) => {
      setTimeout(() => resolve(this.friends.slice()), 200)
    })
  }

  getPhoto (userId) {
    return new Promise((resolve, reject) => {
      setTimeout(() => resolve(this.photo), 200)
    })

  }

  throwError () {
    return new Promise((resolve, reject) => {
      setTimeout(() => reject(new Error('Intentional Error')), 200)
    })
  }

}
```

## 第一次尝试：promise嵌套

首先来看看嵌套的promise写法：
```javascript
const api = new api();
let user, friends, photo;
api.getUser().then(returnedUser => {
  user = returnedUser;

  api.getFriends(user.id).then(returnedFriends => {
    friend = returnedFriends;

    api.getPhoto(user.id).then(photo => {
      photo = photo;
    })

  })
})
```
这不是一个好的实现，本质上跟回调地狱差别不大，只是将回调函数变为了promise，随着异步操作的增多，代码会变得非常冗长而且具有很深的嵌套。更糟糕的是，所有的promise都没有对异常进行捕获和抛出，一旦某个promise失败，外部根本无法感知。

## 第二次尝试：promise链

在前面的[Promise使用基础](https://insaneblue.github.io/2018/04/16/Promise%E4%BD%BF%E7%94%A8%E5%9F%BA%E7%A1%80/)中有提到过promise的最佳实践，那么现在就来试试吧。

```javascript
const api = new api();
let user, friends, photo;
api.getUser()
  .then(returnedUser => {
    user = returnedUser;
    return api.getFriends(user.id)
  })
  .then(returnedFriends => {
    friend = returnedFriends;
    return api.getPhoto(user.id);
  })
  .then(photo => {
    photo = photo;
  })
```

通过promise的链式调用，能够将异步操作按顺序一步一步的执行，并且能够将数据依次传递到下一个then方法中以供下一个操作使用。这样的结构使得代码更加清晰易读，而且在链的尾部还可以添加catch方法来对链中的异常和错误进行捕获，可以说是非常好的改善。
但是这样的写法仍然有些冗余啰嗦，所以就有了第三次尝试。

## 第三次尝试：Async/Await

```javascript
async function getFriend() {
  const api = new Api();
  const user = await api.getUser();
  const friend = await api.getFriends(user.id);
  const photo = await api.getPhoto(user.id);
}
```
这样的写法跟同步代码非常相似，既没有长长的promise链，也没有了嵌套，可读性大大提高了。