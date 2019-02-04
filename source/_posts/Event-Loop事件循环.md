---
title: Event Loop事件循环
date: 2019-02-04 15:00:38
tags: [javascript]
categories: [javascript]
---

Event Loop就是事件循环，是指浏览器或Node的一种解决javaScript单线程运行时不会阻塞的一种机制，也就是我们经常使用异步的原理。

<!--more-->

## Event Loop

在JavaScript中，任务被分为两种，一种宏任务（MacroTask）也叫Task，一种叫微任务（MicroTask）。

+ MacroTask（宏任务）

  script全部代码、setTimeout、setInterval、setImmediate（浏览器暂时不支持，只有IE10支持，具体可见[MDN](https://developer.mozilla.org/zh-CN/docs/Web/API/Window/setImmediate) ）、I/O、UI Rendering。

+ MicroTask（微任务）

  Process.nextTick（Node独有）、Promise、Object.observe(废弃)、MutationObserver(详见[这里](http://javascript.ruanyifeng.com/dom/mutationobserver.html))

## 浏览器中的Event Loop

Javascript 有一个 main thread 主线程和 call-stack 调用栈(执行栈)，所有的任务都会被放到调用栈等待主线程执行。

### JS调用栈

JS调用栈采用的是后进先出的规则，当函数执行的时候，会被添加到栈的顶部，当执行栈执行完成后，就会从栈顶移出，直到栈内被清空。

### 同步任务和异步任务

Javascript单线程任务被分为同步任务和异步任务，同步任务会在调用栈中按照顺序等待主线程依次执行，异步任务会在异步任务有了结果后，将注册的回调函数放入任务队列中等待主线程空闲的时候（调用栈被清空），被读取到栈内等待主线程的执行。

{% asset_img p1.webp.jpg %}


任务队列Task Queue，即队列，是一种先进先出的一种数据结构

{% asset_img p2.webp.jpg %}


### 事件循环的进程模型

+ 选择当前要执行的任务队列，选择任务队列中最先进入的任务，如果任务队列为空即null，则执行跳转到微任务（MicroTask）的执行步骤。

+ 将事件循环中的任务设置为已选择任务。

+ 执行任务。

+ 将事件循环中当前运行任务设置为null。

+ 将已经运行完成的任务从任务队列中删除。

+ microtasks步骤：进入microtask检查点。

+ 更新界面渲染。

+ 返回第一步。

### 执行进入microtask检查点时，用户代理会执行以下步骤

+ 设置microtask检查点标志为true。

+ 当事件循环microtask执行不为空时：选择一个最先进入的microtask队列的microtask，将事件循环的microtask设置为已选择的microtask，运行microtask，将已经执行完成的microtask为null，移出microtask中的microtask。

+ 清理IndexDB事务

+ 设置进入microtask检查点的标志为false。

{% asset_img p3.gif %}

执行栈在执行完同步任务后，查看执行栈是否为空，如果执行栈为空，就会去检查微任务(microTask)队列是否为空，如果为空的话，就执行Task（宏任务），否则就一次性执行完所有微任务。
每次单个宏任务执行完毕后，检查微任务(microTask)队列是否为空，如果不为空的话，会按照先入先出的规则全部执行完微任务(microTask)后，设置微任务(microTask)队列为null，然后再执行宏任务，如此循环。

## 例子
```javascript
console.log('script start');

setTimeout(function() {
  console.log('setTimeout');
}, 0);

Promise.resolve().then(function() {
  console.log('promise1');
}).then(function() {
  console.log('promise2');
});

console.log('script end');
```

### 第一次执行：

```
Tasks：run script、 setTimeout callback

Microtasks：Promise then

JS stack: script
Log: script start、script end。
```
执行同步代码，将宏任务（Tasks）和微任务(Microtasks)划分到各自队列中。

### 第二次执行：
```
Tasks：run script、 setTimeout callback
Microtasks：Promise2 then

JS stack: Promise2 callback
Log: script start、script end、promise1、promise2
```
执行宏任务后，检测到微任务(Microtasks)队列中不为空，执行Promise1，执行完成Promise1后，调用Promise2.then，放入微任务(Microtasks)队列中，再执行Promise2.then。

### 第三次执行：
```
Tasks：setTimeout callback

Microtasks：

JS stack: setTimeout callback
Log: script start、script end、promise1、promise2、setTimeout
```
当微任务(Microtasks)队列中为空时，执行宏任务（Tasks），执行setTimeout callback，打印日志。

### 第四次执行：
```
Tasks：setTimeout callback

Microtasks：

JS stack:
Log: script start、script end、promise1、promise2、setTimeout
```
清空Tasks队列和JS stack。

[这里](https://jakearchibald.com/2015/tasks-microtasks-queues-and-schedules/)有一个更好的地方，可以查看以上执行的帧动画。

## 再举个例子
```javascript
console.log('script start')

async function async1() 
  await async2()
  console.log('async1 end')
}

async function async2() {
  console.log('async2 end')
}

async1()

setTimeout(function() {
  console.log('setTimeout')
}, 0)

new Promise(resolve => {
  console.log('Promise')
  resolve()
})
  .then(function() {
    console.log('promise1')
  })
  .then(function() {
    console.log('promise2')
  })

console.log('script end')
```
async/await 在底层转换成了 promise 和 then 回调函数，其实就是 promise 的语法糖。
每次我们使用 await, 解释器都创建一个 promise 对象，然后把剩下的 async 函数中的操作放到 then 回调函数中。

### 关于73以下版本和73版本的区别

+ 在73版本以下，先执行promise1和promise2，再执行async1。

+ 在73版本，先执行async1再执行promise1和promise2。

### 在73以下版本中

+ 首先，传递给 await 的值被包裹在一个 Promise 中。然后，处理程序附加到这个包装的 Promise，以便在 Promise 变为 fulfilled 后恢复该函数，并且暂停执行异步函数，一旦 promise 变为 fulfilled，恢复异步函数的执行。

+ 每个 await 引擎必须创建两个额外的 Promise（即使右侧已经是一个 Promise）并且它需要至少三个 microtask 队列 ticks（tick为系统的相对时间单位，也被称为系统的时基，来源于定时器的周期性中断（输出脉冲），一次中断表示一个tick，也被称做一个“时钟滴答”、时标。）。

### 例如
```javascript
async function f() {
  await p
  console.log('ok')
}
```
简化理解为：
```javascript
function f() {
  return RESOLVE(p).then(() => {
    console.log('ok')
  })
}
```
+ 如果 RESOLVE(p) 对于 p 为 promise 直接返回 p 的话，那么 p的 then 方法就会被马上调用，其回调就立即进入 job 队列。

+ 而如果 RESOLVE(p) 严格按照标准，应该是产生一个新的 promise，尽管该 promise确定会 resolve 为 p，但这个过程本身是异步的，也就是现在进入 job 队列的是新 promise 的 resolve过程，所以该 promise 的 then 不会被立即调用，而要等到当前 job 队列执行到前述 resolve 过程才会被调用，然后其回调（也就是继续 await 之后的语句）才加入 job 队列，所以时序上就晚了。