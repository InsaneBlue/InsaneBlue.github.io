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

# 异步操作

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

# 使用方法

这里用一个简单的例子来说明使用方法。例如要实现指定多少毫秒后输出一个值，那么可以这样写：

```javascript
function timeout(ms) {
  return new Promise((resolve) => {
    setTimeout(resolve, ms);
  });
}

async function asyncPrint(value, ms) {
  await timeout(ms);
  console.log(value);
}

asyncPrint('hello world', 50);
```

async函数调用时的返回值是一个promise，那么也可以这样写：
```javascript
async function timeout(ms) {
  await new Promise((resolve) => {
    setTimeout(resolve, ms);
  });
}

async function asyncPrint(value, ms) {
  await timeout(ms);
  console.log(value);
}

asyncPrint('hello world', 50);
```

# 语法及错误处理


## 语法

1. async函数的返回值
async函数返回一个 Promise 对象，如果函数内部return语句返回的值不是一个promise，那么会被包装成一个新的promise，并把返回值当成参数传递给下一个then方法。
```javascript
async function f() {
  return 'hello world';
}

f().then(v => console.log(v)); // "hello world"
```
async函数内部抛出错误，会导致返回的 Promise 对象变为reject状态。抛出的错误对象会被catch方法回调函数接收到。
```javascript
async function f() {
  throw new Error('出错了');
}

f().then(v => console.log(v))
  .catch(error => console.log(error))
// Error: 出错了
```
async函数返回的 Promise 对象，必须等到内部所有await命令后面的 Promise 对象执行完，才会发生状态改变，除非遇到return语句或者抛出错误。也就是说，只有async函数内部的异步操作执行完，才会执行then方法指定的回调函数。简单来说，函数执行到await命令时会停止，直到await后面的promise状态发生改变之后才会继续往下执行，所以async函数返回的promise对象状态要发生变化，内部的异步操作肯定已经全部完成了。

```javascript
async function getTitle(url) {
  let response = await fetch(url);
  let html = await response.text();
  return html.match(/<title>([\s\S]+)<\/title>/i)[1];
}
getTitle('https://tc39.github.io/ecma262/').then(console.log)
// "ECMAScript 2017 Language Specification"
```

2. await命令
await命令后面如果是一个Promise 对象，那么返回该对象的结果。如果不是 Promise 对象，就直接返回对应的值。

```javascript
async function f() {
  // 等同于
  // return 123;
  return await 123;
}

f().then(v => console.log(v))
// 123
```

任何一个await语句后面的 Promise 对象变为reject状态，那么整个async函数都会中断执行。
```javascript
async function f() {
  await Promise.reject('出错了');
  await Promise.resolve('hello world'); // 不会执行
}
```
有时，我们希望即使前一个异步操作失败，也不要中断后面的异步操作。这时可以将第一个await放在try...catch结构里面，这样不管这个异步操作是否成功，第二个await都会执行。

```javascript
async function f() {
  try {
    await Promise.reject('出错了');
  } catch(e) {
  }
  return await Promise.resolve('hello world');
}

f()
.then(v => console.log(v))
// hello world
// 或者是在await后面的 Promise 对象再跟一个catch方法，处理前面可能出现的错误。
async function f() {
  await Promise.reject('出错了')
    .catch(e => console.log(e));
  return await Promise.resolve('hello world');
}

f()
.then(v => console.log(v))
// 出错了
// hello world
```

## 错误处理

async的语法规则比较简单，写出来的代码可读性也非常高，其难点主要在错误处理机制。