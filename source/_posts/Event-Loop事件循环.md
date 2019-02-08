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

### 谷歌（金丝雀）73版本中

+ 使用对PromiseResolve的调用来更改await的语义，以减少在公共awaitPromise情况下的转换次数。

+ 如果传递给 await 的值已经是一个 Promise，那么这种优化避免了再次创建 Promise 包装器，在这种情况下，我们从最少三个 microtick 到只有一个 microtick。

### 详细过程：

#### 73以下版本
+ 首先，打印script start，调用async1()时，返回一个Promise，所以打印出来async2 end。

+ 每个 await，会新产生一个promise,但这个过程本身是异步的，所以该await后面不会立即调用。

+ 继续执行同步代码，打印Promise和script end，将then函数放入微任务队列中等待执行。

+ 同步执行完成之后，检查微任务队列是否为null，然后按照先入先出规则，依次执行。

+ 然后先执行打印promise1,此时then的回调函数返回undefinde，此时又有then的链式调用，又放入微任务队列中，再次打印promise2。

+ 再回到await的位置执行返回的 Promise 的 resolve 函数，这又会把 resolve 丢到微任务队列中，打印async1 end。

+ 当微任务队列为空时，执行宏任务,打印setTimeout。

#### 谷歌（金丝雀73版本）

+ 如果传递给 await 的值已经是一个 Promise，那么这种优化避免了再次创建 Promise 包装器，在这种情况下，我们从最少三个 microtick 到只有一个 microtick。

+ 引擎不再需要为 await 创造 throwaway Promise - 在绝大部分时间。

+ 现在 promise 指向了同一个 Promise，所以这个步骤什么也不需要做。然后引擎继续像以前一样，创建 throwaway Promise，安排 PromiseReactionJob 在 microtask 队列的下一个 tick 上恢复异步函数，暂停执行该函数，然后返回给调用者。

## NodeJS的Event Loop

{% asset_img p4.jpg %}

Node中的Event Loop是基于libuv实现的，而libuv是 Node 的新跨平台抽象层，libuv使用异步，事件驱动的编程方式，核心是提供i/o的事件循环和异步回调。libuv的API包含有时间，非阻塞的网络，异步文件操作，子进程等等。 Event Loop就是在libuv中实现的。

{% asset_img p5.jpg %}

### Node的Event loop一共分为6个阶段，每个细节具体如下：

+ timers: 执行setTimeout和setInterval中到期的callback。

+ pending callback: 上一轮循环中少数的callback会放在这一阶段执行。

+ idle, prepare: 仅在内部使用。

+ poll: 最重要的阶段，执行pending callback，在适当的情况下会阻塞在这个阶段。

+ check: 执行setImmediate(setImmediate()是将事件插入到事件队列尾部，主线程和事件队列的函数执行完成之后立即执行setImmediate指定的回调函数)的callback。

+ close callbacks: 执行close事件的callback，例如socket.on('close'[,fn])或者http.server.on('close, fn)。

### timers

执行setTimeout和setInterval中到期的callback，执行这两者回调需要设置一个毫秒数，理论上来说，应该是时间一到就立即执行callback回调，但是由于system的调度可能会延时，达不到预期时间。
以下是官网文档解释的例子：

```javascript
const fs = require('fs');

function someAsyncOperation(callback) {
  // Assume this takes 95ms to complete
  fs.readFile('/path/to/file', callback);
}

const timeoutScheduled = Date.now();

setTimeout(() => {
  const delay = Date.now() - timeoutScheduled;
  console.log(`${delay}ms have passed since I was scheduled`);
}, 100);

// do someAsyncOperation which takes 95 ms to complete
someAsyncOperation(() => {
  const startCallback = Date.now();
  // do something that will take 10ms...
  while (Date.now() - startCallback < 10) {
    // do nothing
  }
});
```
当进入事件循环时，它有一个空队列（fs.readFile()尚未完成），因此定时器将等待剩余毫秒数，当到达95ms时，fs.readFile()完成读取文件并且其完成需要10毫秒的回调被添加到轮询队列并执行。
当回调结束时，队列中不再有回调，因此事件循环将看到已达到最快定时器的阈值，然后回到timers阶段以执行定时器的回调。

在此示例中，您将看到正在调度的计时器与正在执行的回调之间的总延迟将为105毫秒。

以下是我测试时间：

{% asset_img p6.jpg %}

### pending callbacks

此阶段执行某些系统操作（例如TCP错误类型）的回调。 例如，如果TCP socket ECONNREFUSED在尝试connect时receives，则某些* nix系统希望等待报告错误。 这将在pending callbacks阶段执行。

### poll

该poll阶段有两个主要功能：

  + 执行I/O回调。
  + 处理轮询队列中的事件。

当事件循环进入poll阶段并且在timers中没有可以执行定时器时，将发生以下两种情况之一

  + 如果poll队列不为空，则事件循环将遍历其同步执行它们的callback队列，直到队列为空，或者达到system-dependent（系统相关限制）。

如果poll队列为空，则会发生以下两种情况之一

  + 如果有setImmediate()回调需要执行，则会立即停止执行poll阶段并进入执行check阶段以执行回调。

  + 如果没有setImmediate()回到需要执行，poll阶段将等待callback被添加到队列中，然后立即执行。

当然设定了 timer 的话且 poll 队列为空，则会判断是否有 timer 超时，如果有的话会回到 timer 阶段执行回调。

### check

此阶段允许人员在poll阶段完成后立即执行回调。
如果poll阶段闲置并且script已排队setImmediate()，则事件循环到达check阶段执行而不是继续等待。

setImmediate()实际上是一个特殊的计时器，它在事件循环的一个单独阶段运行。它使用libuv API来调度在poll阶段完成后执行的回调。

通常，当代码被执行时，事件循环最终将达到poll阶段，它将等待传入连接，请求等。
但是，如果已经调度了回调setImmediate()，并且轮询阶段变为空闲，则它将结束并且到达check阶段，而不是等待poll事件。

```javascript
console.log('start')

setTimeout(() => {
  console.log('timer1')
  Promise.resolve().then(function() {
    console.log('promise1')
  })
}, 0)

setTimeout(() => {
  console.log('timer2')
  Promise.resolve().then(function() {
    console.log('promise2')
  })
}, 0)

Promise.resolve().then(function() {
  console.log('promise3')
})

console.log('end')
```

如果node版本为v11.x， 其结果与浏览器一致。

```javascript 
start
end
promise3
timer1
timer2
promise1
promise2
```

具体详情可以查看《又被node的eventloop坑了，这次是node的锅》(https://juejin.im/post/5c3e8d90f265da614274218a)。

如果v10版本上述结果存在两种情况：

  + 如果time2定时器已经在执行队列中了，那么执行结果与上面结果相同。

  + 如果time2定时器没有在执行对列中，执行结果为

```javascript
start
end
promise3
timer1
promise1
timer2
```

具体情况可以参考poll阶段的两种情况。

从下图可能更好理解：

{% asset_img p7.git %}

### setImmediate() 的setTimeout()的区别

setImmediate和setTimeout()是相似的，但根据它们被调用的时间以不同的方式表现。

  + setImmediate()设计用于在当前poll阶段完成后check阶段执行脚本 。

  + setTimeout() 安排在经过最小（ms）后运行的脚本，在timers阶段执行。

### 举个例子

```javascript
setTimeout(() => {
  console.log('timeout');
}, 0);

setImmediate(() => {
  console.log('immediate');
});
```

执行定时器的顺序将根据调用它们的上下文而有所不同。 如果从主模块中调用两者，那么时间将受到进程性能的限制。
其结果也不一致
如果在I / O周期内移动两个调用，则始终首先执行立即回调：

```javascript
const fs = require('fs');

fs.readFile(__filename, () => {
  setTimeout(() => {
    console.log('timeout');
  }, 0);
  setImmediate(() => {
    console.log('immediate');
  });
});
```

其结果可以确定一定是immediate => timeout。
主要原因是在I/O阶段读取文件后，事件循环会先进入poll阶段，发现有setImmediate需要执行，会立即进入check阶段执行setImmediate的回调。

然后再进入timers阶段，执行setTimeout，打印timeout。
```javascript
┌───────────────────────────┐
┌─>│ timers │
│ └─────────────┬─────────────┘
│ ┌─────────────┴─────────────┐
│ │ pending callbacks │
│ └─────────────┬─────────────┘
│ ┌─────────────┴─────────────┐
│ │ idle, prepare │
│ └─────────────┬─────────────┘ ┌───────────────┐
│ ┌─────────────┴─────────────┐ │ incoming: │
│ │ poll │<─────┤ connections, │
│ └─────────────┬─────────────┘ │ data, etc. │
│ ┌─────────────┴─────────────┐ └───────────────┘
│ │ check │
│ └─────────────┬─────────────┘
│ ┌─────────────┴─────────────┐
└──┤ close callbacks │
   └───────────────────────────┘
```

### Process.nextTick()

process.nextTick()虽然它是异步API的一部分，但未在图中显示。这是因为process.nextTick()从技术上讲，它不是事件循环的一部分。

process.nextTick()方法将 callback 添加到next tick队列。 一旦当前事件轮询队列的任务全部完成，在next tick队列中的所有callbacks会被依次调用。

换种理解方式：

当每个阶段完成后，如果存在 nextTick 队列，就会清空队列中的所有回调函数，并且优先于其他 microtask 执行。

### 例子

```javascript
let bar;

setTimeout(() => {
  console.log('setTimeout');
}, 0)

setImmediate(() => {
  console.log('setImmediate');
})

function someAsyncApiCall(callback) {
  process.nextTick(callback);
}

someAsyncApiCall(() => {
  console.log('bar', bar); // 1
});

bar = 1;
```

在NodeV10中上述代码执行可能有两种答案，一种为：

```javascript
bar 1
setTimeout
setImmediate
```

另一种为：

```javascript
bar 1
setImmediate
setTimeout
```
无论哪种，始终都是先执行process.nextTick(callback)，打印bar 1。

**转载自奇舞周刊的[这篇文章](https://mp.weixin.qq.com/s?__biz=MzA5NzkwNDk3MQ==&mid=2650589062&idx=1&sn=4483d25d350b5b7433b807c8f0150228&chksm=8891d7a2bfe65eb44a5bd29956bda5701da182be3372947d408e70be9771221619f1841d32fa&scene=0#rd)**
