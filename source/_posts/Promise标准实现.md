---
title: Promise标准实现
date: 2018-04-12 23:13:23
tags: promise
categories: promise
---

# Promise标准实现

## 前言

我在想去自己实现一个Promise类库的时候，首先会去找一些比较简洁又符合标准的一些相关实现，去分析其源码，然后结合几种实现的优点总结出自己的版本，站在巨人的肩膀上让我直接取道直径，快速的实现了我的目标，在这里非常感谢前辈们的努力和给我们留下的宝贵知识财富。

从标准中寻找蛛丝马迹 (以下所说的 `标准` 均以 [Promise/A+](https://promisesaplus.com/) 做为参考)，我们将依据标准，编写一个可通过标准测试的Promise类库。

<!--more-->

## Promise类的结构

标准中规定：

1. Promise对象初始状态为 `Pending`，在被 `resolve` 或 `reject` 时，状态变为 `Fulfilled` 或 `Rejected`
2. resolve接收成功的数据，reject接收失败或错误的数据
3. Promise对象必须有一个 `then` 方法，且只接受两个可函数参数 `onFulfilled`、`onRejected`

由以上标准就容易就能实现这个类的大致结构

```javascript
// resolver为 function(resolve, reject){ ... }
function Promise(resolver){
  if(resolver && typeof resolver !== 'function'){ throw new Error('Promise resolver is not a function') }
  // 当前promise对象的状态
  this.state = PENDING;
  // 当前promise对象的数据（成功或失败）
  this.data = UNDEFINED;
  // 当前promise对象注册的回调队列
  this.callbackQueue=[];
  // 执行resove()或reject()方法
  if(resolver) executeResolver.call(this, resolver);
}
Promise.prototype.then = function(){}

```

所以，一个 `Promise构造函数` 和`一个实例方法then` 就是Promise的核心的了，其它的都是Promise的语法糖或者说是扩展。

## 构造器的初始化

使用 `new Promise(function(resolve, reject){...} )`实例化 Promise 时，去改变promise的状态，是执行 resolve() 或 reject()方法，那么，resolver的两个参数分别是成功的操作函数和失败的操作函数。

### executeResolver

这里将上面执行 `resolver` 的方法抽象出来，内部再将 `resovle` 和 `reject` 两个参数包装成 成功和失败的回调。

```javascript
// 用于执行 new Promise(function(resolve, reject){}) 中的resove或reject方法
function executeResolver(resolver){
 // [标准 2.3.3.3.3] 如果resove()方法多次调用，只响应第一次，后面的忽略
  var called = false, _this = this;
  function onError(value) {
    if (called) { return; }
    called = true;
    // [标准 2.3.3.3.2] 如果是错误 使用reject方法
    executeCallback.bind(_this)('reject', value);
  }
  function onSuccess(value) {
    if (called) { return; }
    called = true;
    // [标准 2.3.3.3.1] 如果是成功 使用resolve方法
    executeCallback.bind(_this)('resolve', value);
  }

  // 使用try...catch执行
  // [标准 2.3.3.3.4] 如果调用resolve()或reject()时发生错误，则将状态改成rejected，并将错误reject出去
  try{
    resolver(onSuccess, onError);
  }catch(e){
    onError(e);
  }
}

```

### executeCallback

因为执行 `resolved()` 或 `reject()` 内部主要作用是更改当前实例的状态为 `rejected`或 `resolved`，然后执行当前实例 `then()` 中注册的 成功或失败的回调函数， 功能有相似的地方，所以做成方法直接调用

```javascript
// 用于执行成功或失败的回调 new Promise((resolve, reject) => { resolve(1)或 reject(1) })
function executeCallback(type, x){
    var isResolve = type === 'resolve', thenable;

    // [标准 2.3.3] 如果x是一个对象或一个函数
    if(isResolve && (typeof x === 'object' || typeof x === 'function')){
	  //[标准 2.3.3.2]
      try{
        thenable = getThen(x);
      }catch(e){
        return executeCallback.bind(this)('reject', e);
      }
    }
    if(isResolve && thenable){
      executeResolver.bind(this)(thenable);
    }else{
      //[标准 2.3.4]
      this.state = isResolve ? RESOLVED : REJECTED;
      this.data = x;
      this.callbackQueue && this.callbackQueue.length && this.callbackQueue.forEach(v => v[type](x));
    }
    return this;
  }

```

### getThen

用于判断是否是thenable对象，如果是，则返回一个执行thenable中then方法的函数

```javascript
function getThen(obj){
  var then = obj && obj.then;
  if (obj && typeof obj === 'object' && typeof then === 'function') {
   return function appyThen() {
     then.apply(obj, arguments);
   };
  }
}

```

到这里，Promise实例初始化的处理逻辑就完成了

执行下面这段代码将不会有任何输出，但是promise的状态发生了改变，也就是 `this.status` 为 `resolved`，`this.data` 为 `'ok'`

```javascript
new Promise(resolve=>resolve('ok'))

```

至于如何在 `resolve()` 调用之后，去执行 `then` 方法注册的 `onResolved` 回调，下面继续分析

## then方法

标准中规定：

1. `then` 方法必须返回一个新的 `Promise实例`（ES6中的标准，Promise/A+中没有明确说明）
2. 为了保证 `then`中回调的执行顺序，`onFulfilled` 或 `onRejected` 必须异步调用

对应标准看具体实现

```javascript
Promise.prototype.then = function(onResolved, onRejected){
  //[标准 2.2.1 - 2.2.2] 状态已经发生改变并且参数不是函数时，则忽略
  if (typeof onResolved !== 'function' && this.state === RESOLVED ||
      typeof onRejected !== 'function' && this.state === REJECTED) {
    return this;
  }

  // 实例化一个新的Promise对象
  var promise = new this.constructor();

  // 一般情况下，状态发生改变时，走这里
  if(this.state !== PENDING){
    var callback = this.state === RESOLVED ? onResolved : onRejected;
    // 将上一步 resolve(value)或rejecte(value) 的 value 传递给then中注册的 callback
    // [标准 2.2.4] 异步调用callback
    executeCallbackAsync.bind(promise)(callback, this.data);
  }else{
    // var promise = new Promise(resolve=>resolve(1)); promise.then(...); promise.then(...); ...
    // 一个实例执行多次then, 这种情况会走这里 [标准 2.2.6]
    this.callbackQueue.forEach(v => v[type](x));
  }

  // 返回新的实例 [标准 2.2.7]
  return promise;
}

```

### executeCallbackAsync

上面将异步调用callback的逻辑抽象成了一个方法`executeCallbackAsync` ，这个方法主要功能是安全的执行callback方法：

- 如果出错，则自动调用 `reject(reason)` 方法并更改状态为 `rejected`，传递错误数据给当前实例then方法中注册的`onRejected` 回调
- 如果成功，则自动调用 `resolve(value)`方法并更改状态为`resolved`，传递数据给当前实例then方法中注册的 `onResolved` 回调

```javascript
// 用于异步执行 .then(onResolved, onRejected) 中注册的回调
function executeCallbackAsync(callback, value){
  var _this = this;
  setTimeout(function(){
    var res;
    try{
      res = callback(value);
    }catch(e){
      return executeCallback.bind(_this)('reject', e);
    }

    if(res !== _this){
      return executeCallback.bind(_this)('resolve', res);
    }else{
      return executeCallback.bind(_this)('reject', new TypeError('Cannot resolve promise with itself'));
    }
  }, 1)
}

```

**注意**
这里最好不要用 `setTimeout` ，使用 `setTimeout` 可以异步执行回调，但其实并不是真正的异步线程，而是利用了浏览器的 `Event Loop` 机制去触发执行回调，而浏览器的事件轮循时间间隔是 `4ms` ，所以连接的调用 `setTimeout` 会有 `4ms` 的时间间隔，而在`Nodejs` 中的 `Event Loop` 时间间隔是 `1ms`，所以会产生一定的延迟，如果promise链比较长，延迟就会越明显，这里可以引入NPM上的 `immediate` 模块来异步无延迟的执行回调。

### CallbackItem

上面then中对于回调的处理，使用了一个回调对象来管理注册的回调，将回调按顺序添加至 `callbackQueue` 队列中，调用时，依次调用。

```javascript
// 用于注册then中的回调 .then(resolvedFn, rejectedFn)
function CallbackItem(promise, onResolved, onRejected){
  this.promise = promise;
  // 为了保证在promise链中，resolve或reject的结果可以一直向后传递，可以默认给then添加resolvedFn和rejectedFn
  this.onResolved = typeof onResolved === 'function' ? onResolved : function(v){return v};
  this.onRejected = typeof onRejected === 'function' ? onRejected : function(v){throw v};
}
CallbackItem.prototype.resolve = function(value){
//调用时异步调用 [标准 2.2.4]
  executeCallbackAsync.bind(this.promise)(this.onResolved, value);
}
CallbackItem.prototype.reject = function(value){
//调用时异步调用 [标准 2.2.4]
  executeCallbackAsync.bind(this.promise)(this.onRejected, value);
}

```

## catch方法

由于catch方法是then(null, onRejected)的语法糖，所以这里也很好实现

```javascript
Promise.prototype.catch = function(onRejected){
  return this.then(null, onRejected);
}

```

到这里，我们自定义的Promise的相关代码核心都实现完成了，下面可以测试一下

```javascript
function fn1(){
    var promise = new Promise(resolve=>resolve(1))
    promise.then(res=>console.log(++res))
    promise.then(res=>console.log(++res))
    promise.then(res=>console.log(++res))
}
fn1()
// 2, 2, 2
function fn2(){
    var promise = new Promise((resolve, reject)=>{
        resolve(1)
    })
    .then(res=>(res++, console.log(res), res))
    .then(res=>(res++, console.log(res), res))
    .then(res=>(res++, console.log(res), res))
}
fn2()
// 2, 3, 4

function fn3(){
    return new Promise((resolve,reject)=>reject(999))
}
fn3().catch(err=>console.log(err))
// 999

```

## 添加扩展方法

Promise的扩展方法都是基于Promise的构造函数和then方法的实现，不同的类库实现方式不同，部分API意义也不同

### all

用于并行执行promise组成的数组（数组中可以不是Promise对象，在调用过程中会使用 `Promise.resolve(value)` 转换成Promise对象），如果全部成功则获得成功的结果组成的数组对象，如果失败，则获得失败的信息，返回一个新的Promise对象

```javascript
Promise.all = function(iterable){
 var _this = this;
 return new this(function(resolve, reject){
   if(!iterable || !Array.isArray(iterable)) return reject( new TypeError('must be an array') );
   var len = iterable.length;
   if(!len) return resolve([]);

   var res = Array(len), called=false;

   iterable.forEach(function(v, i){
     (function(i){
       _this.resolve(v).then(function(value){
         res[i]=value;
         if(++counter===len && !called){
           called = true;
           return resolve(res)
         }
       }, function(err){
         if(!called){
           called = true;
           return reject(err);
         }
       })
     })(i)
   })
 })
}

```

使用方式

```javascript
function fn1(){
    return new Promise(resolve => setTimeout(()=>resolve(1), 3000))
}
function fn2(){
    return new Promise(resolve => setTimeout(()=>resolve(2), 2000))
}
Promise.all([fn1(), fn2()]).then(res=>console.log(res), err=>console.log(err))
// [1, 2]

```

这里的实现，要注意一点的是，**即要保证全部成功，又要保证按数组里原来的顺序返回结果**，在一开始的实现里，我并没有考虑到这个问题，所以最初的实现是没有添加闭包的，那么结果就是数组里的promise谁先成功，谁的结果就占据了第一个位置，就算这个promise是数组的最后一个

最初的 **错误** 实现

```javascript
var res = Array(len), counter = 0, called=false;
iterable.forEach(function(v){
    _this.resolve(v).then(function(value){
      res[counter] = value;
      if(++counter===len && !called){
        called = true;
        return resolve(res)
      }
    }, function(err){
      if(!called){
        called = true;
        return reject(err);
      }
    })
})

```

这样导致的结果就是，返回的结果数组与原来的数组不能一一匹配，上面的测试就会返回 [2, 1]

### race

用于并行执行promise组成的数组（数组中可以不是Promise对象，在调用过程中会使用 `Promise.resolve(value)` 转换成Promise对象），如果某个promise的状态率先改变，就获得改变的结果，返回一个新的Promise对象

```javascript
Promise.race = function(iterable){
  var _this = this;
  return new this(function(resolve, reject){
    if(!iterable || !Array.isArray(iterable)) return reject( new TypeError('must be an array') );
    var len = iterable.length;
    if(!len) return resolve([]);

    var called = false;
    iterable.forEach(function(v, i){
      _this.resolve(v).then(function(res){
        if(!called){
          called = true;
          return resolve(res);
        }
      }, function(err){
        if(!called){
          called = true;
          return reject(err);
        }
      })
    })
  })
}

```

因为 `race` 是返回单个的结果，所以不存类似于 `all` 的情况

使用方式

```javascript
function fn1(){
    return new Promise(resolve => setTimeout(()=>resolve(1), 3000))
}
function fn2(){
    return new Promise(resolve => setTimeout(()=>resolve(2), 2000))
}
Promise.race([fn1(), fn2()]).then(res=>console.log(res), err=>console.log(err))
// 2

```

### resolve

用于包装任意对象为promise对象，返回一个新的promise,并且状态是resolved

```javascript
Promise.resolve = function(value){
  if(value instanceof this) return value;
  return executeCallback.bind(new this())('resolve', value);
}

```

### reject

用于包装任意对象为promise对象，返回一个新的promise,并且状态是rejected

```javascript
Promise.reject = function(value){
  if(value instanceof this) return value;
  return executeCallback.bind(new this())('reject', value);
}

```

### wait

用于一个promise任务结束后等待指定的时间再去执行一些操作

```javascript
Promise.prototype.wait = function(ms){
  var P = this.constructor;
  return this.then(function(v){
    return new P(function(resolve, reject){
      setTimeout(function(){ resolve(v); }, ~~ms)
    })
  }, function(r){
    return new P(function(resolve, reject){
      setTimeout(function(){ reject(r); }, ~~ms)
    })
  })
}

```

使用

```javascript
fn1().wait(2000).then(res=>console.log(res),err=>console.log(err))

```

这里考虑到，`wait` 是用于promise实例对象上的，那么为了可以保证链式调用，必须返回一个 `新的promise`，并且上一步的成功和失败的消息不能丢失，继续向后传递，这里只做延迟处理。

### stop

用于中断promise链

通常在 `promise链` 中去reject或throw，或者是异常报错信息，promise内部都会使用 `try...catch` 转换为 `reject` 方法往后传递，无法中断后面的 `then` 或其它方法的执行，那么这里利用，`then` 方法中对状态的要求必须不是 `Pending` 状态的处理才会立即执行回调，在 `promise链` 中返回一个初始状态的 `Promise对象`，便可以中断后面回调的执行。

```javascript
Promise.stop = function(){
  return new this();
}

```

使用

```javascript
Promise
  .resolve(1)
  .then(res=>{
	console.log('发生错误，停止后面的执行')
	return Promise.stop();
  })
  .then(res=>console.log(res))
  .catch(err=>console.log(err))

```

### always

无论成功还是失败最终都会调用 `always` 中注册的回调

```javascript
Promise.prototype.always = function(fn){
  return this.then(function(v){
    return fn(v), v;
  }, function(r){
    throw fn(r), r;
  })
}

```

使用

```javascript
ajaxLoadData()
  .then(res=>console.log(res), err=>console.log(err))
  .always(()=>console.log('关闭loading动画'))

```

### done

由于promise在执行 `resolve` 或 `onResolved` 回调时，使用了`try...catch`，并将错误信息，使用 `reject方法` 传递了出去，但是如果后面没有注册处理reject的回调函数，那么错误信息将无法得到处理，进而消失不见，难以查觉，所以有了 `done` 方法。

`done`方法并不返回promise对象，也就是done之后不能使用 `then`或`catch`了，其主要作用就是用于将 `promise链` 中未捕获的异常信息抛至外层，并不会对错误信息进行处理。

**done方法必须应用于promise链的最后**

```javascript
Promise.prototype.done = function(onResolved, onRejected){
  this.then(onResolved, onRejected).catch(function (error) {
      setTimeout(function () {
          throw error;
      }, 0);
  });
}

```

使用

```javascript
ajaxLoadData()
  .then(res=>{
    return new Promise((resolve,reject)=>reject('未捕获的错误'))
  }, err=>console.log(err))
  .always(()=>console.log('关闭loading动画'))
  .done()//这里会将错误信息 '未捕获的错误' 抛至外层

```

### defer

`Deferred` 的简称，叫延迟对象，其实是 `new Promise()` 的语法糖

与Promise的关系

- Deferred 拥有 Promise
- Deferred 具备对 Promise的状态进行操作的特权方法
- Promise 代表了一个对象，这个对象的状态会在未来改变
- Deferred对象 表示了一个处理没有结束，在状态发生改变时，再使用Promise来处理结果

优缺点

- 不用使用大括号将逻辑包起来，少了一层嵌套
- 但是缺少了Promise的错误处理逻辑

```javascript
Promise.deferred = Promise.defer = function(){
  var dfd = {}
  dfd.promise = new this(function(resolve, reject) {
    dfd.resolve = resolve;
    dfd.reject = reject;
  })
  return dfd
}

```

使用

```javascript
function getURL(URL) {
    var deferred = Promise.deferred;
    var req = new XMLHttpRequest();
    req.open('GET', URL, true);
    req.onload = function () {
        if (req.status === 200) {
            deferred.resolve(req.responseText);
        } else {
            deferred.reject(new Error(req.statusText));
        }
    };
    req.onerror = function () {
        deferred.reject(new Error(req.statusText));
    };
    req.send();
    return deferred.promise;
}

```

### timeout

用于判断某些promise任务是否超时
如一个异步请求，如果超时，取消息请求，提示消息或重新请求

```javascript
Promise.timeout = function(promise, ms){
  return this.race([promise, this.reject().wait(ms)]);
}

```

用法

```javascript
function fn4(){
    return new Promise(resolve=> setTimeout(()=>resolve(1), 3000))
}

Promise
    .timeout(fn4(), 2000)
    .then(res=>console.log(res), err=>console.log('超时'))
// 这里 fn4需要3s执行完成，这里只准在2s内完成，fn4的执行时间就超时了，会输出 `超时`

```

### sequence

用于按顺序执行一系列的promise，接收的**函数数组**，并**不是Promise对象数组**，其中**函数执行时就返回Promise对象**，用于有互相依赖的promise任务

```javascript
Promise.sequence = function(tasks){
    return tasks.reduce(function (prev, next) {
        return prev.then(next).then(function(res){ return res });
    }, this.resolve());
}

```

使用

```javascript
function fn1(){ return new Promise(r=>r(1))}
function fn2(data){ return new Promise(r=>r(1+data))}
function fn3(data){ return new Promise(r=>r(1+data))}

Promise.sequence([fn1,fn2,fn3]).then(res=>console.log(res))
//3

```

## 测试

对于自定义的Promise类库，是否符合 [Promise/A+](https://promisesaplus.com/) 的标准呢？

社区有一个开源的[测试脚本](https://github.com/promises-aplus/promises-tests)
只需两步，就能检验我们的实现是否符合标准了

```javascript
//全局安装
npm i -g promises-aplus-tests
//运行测试
promises-aplus-tests Promise.js

```

下面是我们自定义的Promise类库的测试结果，全部通过

![img](http://7xi480.com1.z0.glb.clouddn.com/%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A7%202016-12-04%20%E4%B8%8A%E5%8D%889.20.57.png)

在一开始运行测试的时候并没有那么顺利

首先要注意的一点是，自己封装的Promise，要提供 `Promise.deferred()` 静态方法，测试脚本是基于这个方法去测试的，一定要有!

```javascript
Promise.deferred = Promise.defer = function(){
  var dfd = {}
  dfd.promise = new Promise(function(resolve, reject) {
    dfd.resolve = resolve;
    dfd.reject = reject;
  })
  return dfd
}

```

在测试过程中，提示错误最多的是在于 `标准 2.3.3` 的实现，代码中的实现一定要严格按照标准来，特别是关于 `thenable的判断` 与`then方法获取`，详情请见 `executeCallback` 的实现

那么这个类库是否可以用于生产环境了呢？

答案是不可以哦，因为测试用例只测试了Promise的核心标准，也就是只测试了 `then方法`、`deferred方法` 和 `构造函数`，其它的扩展方法一个都没有测试，所以要用于生产环境，我们还需要单独编写更多的测试用例才可以

当然扩展方法都是可以运行的，只是少了一些容错机制，如果你对这个类库感兴趣，那么也可以尝试使用，一起来帮忙完善这个工具。

到这里终于完成我们的自定义又符合标准的Promise类库了

## 源码地址

[Github MPromise](https://github.com/git-lt/MPromise)
核心代码只有120行，加上扩展230行，应该算是很小的体积了
如果你从这篇文章了解到了你之前未曾知道的Promise相关知识
如果你对这个工具类库感兴趣
如果你希望以后能看到我更多的分享

那么，希望你能给这个项目一个 `star` ，你的鼓励是我不但前行的动力哦，哈哈

下一篇，我们将根据这个实现结合项目应用场景，实践一下。