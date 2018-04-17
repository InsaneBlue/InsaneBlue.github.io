---
title: Promise使用基础
date: 2018-04-16 23:25:47
tags: promise
categories: promise
---

# Promise使用基础

## 前言

前面两篇文章介绍了Promise的原理及标准实现，本篇主要介绍Promise的使用基础和各种不同的用法。虽然平时一直都在使用Promise，但是对于其中的一些注意点和最佳实践还有一些不太清晰的地方，所以在这里补上一篇使用基础。

<!--more-->

## Promise 规范

这里只列举规范中的大致内容，详细内容请查看 [Promises/A+ 中文](https://segmentfault.com/a/1190000002452115) ，这是ES6 Promises的前身，是一个社区规范，它和 ES6 Promises 有很多共通的内容。

1. **状态** `Promise` 的初始状态是 `Pending` ，状态只能被转换为`（Resolved）Fulfilled`或`Rejected`，状态的转换不可逆。
2. **then** 必须有 `then` 方法，接收两个可选函数参数`onFulfilled`、`onRejected`，`then` 方法必须返回一个新的 `Promise` 对象，为了保证 `then` 中回调的执行顺序，回调必须使用异步执行。
3. **兼容** 不同的 `Promise` 的实现必须可以互相调用

具体标准的实现在 [上一篇](https://insaneblue.github.io/2018/04/12/Promise%E6%A0%87%E5%87%86%E5%AE%9E%E7%8E%B0/) 中有详细说明

## ES6 Promise API

如果你对 Promise的使用 还不是很了解，可参考阅读以下资料：

- [promises-book](http://liubin.org/promises-book/#introduction)(非常推荐大家阅读的资料，对Promise讲解十分细致详尽)
- [阮一峰的Promise科普文](http://es6.ruanyifeng.com/#docs/promise)

这里只对ES6 Promise API做简要说明

### 实例方法

- `.then(resolvedFn, rejectFn)`
  为Promise实例添加状态改变时的回调，返回值是一个 **新的Promise实例**
- `.catch()`
  是 `.then(null, rejectFn)` 的语法糖，返回值也是一个 **新的Promise对象**
  Promise对象的错误具有冒泡性质，错误会不断的向后传递，直到 `.catch()` 捕获
  正因为 `then` 和 `catch` 返回的都是 `Promise` 对象，所以才可以不断的链式调用

### 静态方法

- `Promise.resolve()`

1. 将现有对象转换为Promise对象
2. 如果参数是promise实例，则直接返回这个实例
3. 如果参数是thenabled对象（有then方法的对象），则先将其转换为promise对象，然后立即执行这个对象的then方法
4. 如果参数是个原始值，则返回一个promise对象，状态为resolved，这个原始值会传递给回调
5. 没有参数，直接返回一个resolved的Promise对象

- `Promise.reject()`

  同上，不同的是返回的promise对象的状态为rejected

- `Promise.all()`

1. 接收一个Promise实例的数组或具有Iterator接口的对象，
2. 如果元素不是Promise对象，则使用Promise.resolve转成Promise对象
3. 如果全部成功，状态变为resolved，返回值将组成一个数组传给回调
4. 只要有一个失败，状态就变为rejected，返回值将直接传递给回调
5. `all()` 的返回值也是新的Promise对象

- `Promise.race()`

1. 同上，区别是，只要有一个Promise实例率先发生变化（无论是状态变成resolved还是rejected）都触发then中的回调，返回值将传递给回调
2. `race()`的返回值也是新的Promise对象

## Polyfill和扩展类库

### Polyfill

只需要在浏览器中加载Polyfill类库，就能使用IE10等或者还没有提供对Promise支持的浏览器中使用Promise里规定的方法。

[then/promise](https://github.com/then/promise) 可以直接使用`npm install promise`给项目仓库添加这个依赖，也是符合Promises/A+标准的一个promise库。
[jakearchibald/es6-promise](https://github.com/stefanpenner/es6-promise) 兼容 Promises/A+ 的类库， 它只是 RSVP.js 的一个子集，只实现了Promises 规定的 API。

**注意**
在项目中，有可能两个不同的模块使用的是两个不同的Promise类库，那么在大部分的Promise的实现中，都是遵循 Promise/A+ 标准和兼容ES6 Promise接口的，也是不同的Promise的实现是可以互相调用的，如何调用，将在下面说明。

## 错误用法及误区

### 当作回调来用 Callback Hell

```javascript
loadAsync1().then(function(data1) {
  loadAsync2(data1).then(function(data2) {
    loadAsync3(data2).then(okFn, failFn)
  });
});
```

Promise是用来解决异步嵌套回调的，这种写法虽然可行，但违背了Promise的设计初衷
改成下面的写法，会让结构更加清晰

```javascript
loadAsync1()
	.then(function(data1) {
		return loadAsync2(data1)
	})
	.then(function(data2){
	    return loadAsync3(data2)
	})
	.then(okFn, failFn)
```

### 没有返回值

```javascript
loadAsync1()
	.then(function(data1) {
		loadAsync2(data1)
	})
	.then(function(data2){
	    loadAsync3(data2)
	})
	.then(res=>console.log(res))
```

promise 的神奇之处在于让我们能够在回调函数里面使用 return 和 throw， 所以在then中可以return出一个promise对象或普通的值，也可以throw出一个错误对象，但如果没有任何返回，将默认返回 undefined，那么后面的then中的回调参数接收到的将是undefined，而不是上一个then中内部函数 loadAsync2 执行的结果，后面都将是undefined。

### 没有Catch

```javascript
loadAsync1()
	.then(function(data1) {
		return loadAsync2(data1)
	})
	.then(function(data2){
	    return loadAsync3(data2)
	})
	.then(okFn, failFn)
```

这里的调用，并没有添加catch方法，那么如果中间某个环节发生错误，将不会被捕获，控制台将看不到任何错误，不利于调试查错，所以最好在最后添加catch方法用于捕获错误。

添加catch

```javascript
loadAsync1()
	.then(function(data1) {
		return loadAsync2(data1)
	})
	.then(function(data2){
	    return loadAsync3(data2)
	})
	.then(okFn, failFn)
	.catch(err=>console.log(err))
```

### catch()与then(null, fn)

在有些情况下catch与then(null, fn)并不等同，如下

```javascript
ajaxLoad1()
	.then(res=>{ return ajaxLoad2() })
	.catch(err=> console.log(err))
```

此时，catch捕获的并不是ajaxLoad1的错误，而是ajaxLoad2的错误，所以有时候，两者还是要结合起来使用：

```javascript
ajaxLoad1()
	.then(res=>{ return ajaxLoad2() }, err=>console.log(err))
	.catch(err=> console.log(err))
```

### 断链 The Broken Chain

```javascript
function loadAsyncFnX(){ return Promise.resolve(1); }
function doSth(){ return 2; }

function asyncFn(){
	var promise = loadAsyncFnX()
    promise.then(function(){
		return doSth();
    })
	return promise;
}

asyncFn().then(res=>console.log(res)).catch(err=>console.log(err))
// 1
```

上面这种用法，从执行结果来看，then中回调的参数其实并不是doSth()返回的结果，而是loadAsyncFnX()返回的结果，catch 到的错误也是 loadAsyncFnX()中的错误，所以 doSth() 的结果和错误将不会被后而的then中的回调捕获到，形成了断链，因为 then 方法将返回一个新的Promise对象，而不是原来的Promise对象。

改写如下

```javascript
function loadAsyncFnX(){ return Promise.resolve(1); }
function doSth(){ return 2; }

function asyncFn(){
	var promise = loadAsyncFnX()
    return promise.then(function(){
		return doSth();
    })
}

asyncFn().then(res=>console.log(res)).catch(err=>console.log(err))
// 2
```

### 穿透 Fall Through

```javascript
new Promise(resolve=>resolve(8))
  .then(1)
  .catch(null)
  .then(Promise.resolve(9))
  .then(res=> console.log(res))
// 8
```

这里，如果then或catch接收的不是函数，那么就会发生穿透行为，所以在应用过程中，应该保证then接收到的参数始终是一个函数。

### 长度未知的串行与并行

并行执行

```javascript
getAsyncArr()
	.then(promiseArr=>{
		var resArr = [];
		promiseArr.forEach(v=>{
			v().then(res=> resArr.push(res))
		})
		return resArr;
	})
	.then(res=>console.log(res))
```

使用forEach遍历执行promise，在上面的实现中，第二个then有可能拿到的是空的结果或者不完整的结果，因为，第二个then的回调无法预知 promiseArr 中每一个promise是否都执行完成，那么这里可以使用 Promise.all 结合 map 方法去改善

```javascript
getAsyncArr()
	.then(promiseArr=>{
		return Promise.all(promiseArr);
	})
	.then(res=>console.log(res))
```

如果需要串行执行，那和我们可以利用数据的reduce来处理串行执行

```javascript
var pA = [
	function(){return new Promise(resolve=>resolve(1))},
	function(data){return new Promise(resolve=>resolve(1+data))},
	function(data){return new Promise(resolve=>resolve(1+data))}
]

pA.reduce((prev, next)=>prev.then(next).then(res=>res),Promise.resolve())
.then(res=>console.log(res))
// 3
```

### Promise.resolve的用法

`Promise.reoslve` 有一个作用就是可以将 `thenable` 对象转换为 `promise` 对象。

`thenable` 对象，指的是一个具有 `.then` 方法的对象。
要求是 `thenable` 对象所拥有的 `then` 方法应该和 `Promise` 所拥有的 `then` 方法具有同样的功能和处理过程。
一个标准的 thenable 对象应该是这样的

```javascript
var thenable = {
  then: function(resolve, reject) {
    resolve(42);
  }
};
```

使用 Promise.resolve转换

```javascript
Promise.resolve(thenable).then(function(value) {
  console.log(value);  // 42
});
```

同样具有标准的thenable特性的是 不同的实现Promise标准的类库，所以 ES6 Promise 与 Q 的对象都是可以互相转换的。

jQueyr的defer对象转换为ES6 Promise对象

```javascript
Promise.resolve($.ajax('api/data.json')).then(res=>console.log(res)))
```

但也不是所有thenable对象都能被成功转换，主要看各种类库实现是否遵循 Promise/A+标准，不过此类使用场景并不多，不做深入讨论。

## 最佳实践

1. `then`方法中 永远 `return` 或 `throw`
2. 如果 `promise` 链中可能出现错误，一定要添加 `catch`对链中的错误进行捕获和处理
3. 永远传递函数给 `then` 方法
4. 不要把 `promise` 写成嵌套，尽量写成`promise`链去执行分步的异步操作

经过本篇的对Promise相关知识的理解和学习，基本上对Promise的概念和使用有了比较详细的了解。但是要要深刻理解Promise的原理、更好的去使用它，还得靠大量的练习和思考。