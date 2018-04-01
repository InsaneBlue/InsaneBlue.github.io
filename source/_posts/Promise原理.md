---
title: Promise原理
date: 2018-03-17 21:40:03
tags: promise
---

# Promise原理

## 前言

Promise 是一种对异步操作的封装，可以通过独立的接口添加在异步操作执行成功、失败时执行的方法。主流的规范是 [Promises/A+](http://promisesaplus.com/)。

<!--more-->

## 内部机制

### 基础实现

为了增加代入感和增进理解，先从最基础的使用场景开始探索。例一是用Promise来实现通过异步请求获取用户id，然后做一些处理这样一个简单需求。

```javascript
//例1
function getUserId() {
  return new Promise(function(resolve) {
    // 异步请求
    http.get(url, function(results) {
        resolve(results.id)
    })
  })
}
getUserId().then(function(id) {
  // 一些处理
})
```

`getUserId`方法返回一个`promise`实例，可以通过它的`then`方法注册在`promise`异步操作成功时执行的回调。这种执行方式，使得异步调用变得十分简单顺手。

能够满足这样一种场景的Promise其实并不复杂，下面是最基础的实现：

```javascript
function Promise(fn) {
  var callbacks = [];

  this.then = function (onFulfilled) {
    callbacks.push(onFulfilled);
  };

  function resolve(value) {
    callbacks.forEach(function (callback) {
      callback(value);
    });
  }

  fn(resolve);
}
```

代码很短，实现的逻辑也很简单：

- 调用`then`方法，将异步请求成功后需要执行的回调放入callbacks数组中；
- 创建Promise实例时传入函数被赋予一个函数类型的参数，即`resolve`，用以在合适的时机触发，真正执行的操作是将`callbacks`队列中的回调一一执行；
- ``resolve``接收的参数`value`就是异步操作的结果方便回调使用。

有时需要注册多个回调，也就是一个异步操作后面会有多个操作和处理，这样就需要链式操作：

```javascript
// 例2
getUserId().then(function (id) {
  // 一些处理
}).then(function (id) {
  // 再来一些处理
});
```

那么就需要对上面的代码再小小地改造一下。事实上，这很容易：

```javascript
this.then = function (onFulfilled) {
  callbacks.push(onFulfilled);
  return this;
};
```

### 延时

如果使用过Promise会发现上面的实现还有一些问题。如果promise是同步代码，那么`resolve`会在`then`之前执行，这时`callbacks`队列还是空的，更严重的是，后续注册的回调也不会被执行了：

```javascript
// 例3
function getUserId() {
  return new Promise(function (resolve) {
    resolve(1234);
  });
}

getUserId().then(function (id) {
  // 一些处理
});
```

此外，Promises/A+ 规范明确要求回调需要通过异步方式执行，用以保证一致可靠的执行顺序。为解决这两个问题，可以通过 `setTimeout` 将 `resolve` 中执行回调的逻辑放置到 JS 任务队列末尾：

```javascript
function resolve(value) {
  setTimeout(function () {
    callbacks.forEach(function (callback) {
      callback(value);
    });
  }, 0);
}
```

### 引入状态

虽然引入延时会保证`resolve`执行时`then`方法的回调已经注册完成，但是如果Promise的异步操作已经成功，之后再调用`then`注册回调再也不会执行了，这显然是不符合预期的。

解决这个问题就需要引入规范中的states，即每个Promise存在三个互斥状态pending、fulfilled、rejected，一旦状态转换就不再变化：

{% asset_img promise-states-flow.png %}

代码当然也需要改进：

```javascript
function Promise(fn) {
  var state = 'pending',
      value = null,
      callbacks = [];

  this.then = function (onFulfilled) {
    if (state === 'pending') {
      callbacks.push(onFulfilled);
        return this;
      }
      onFulfilled(value);
      return this;
  };

  function resolve(newValue) {
    value = newValue;
    state = 'fulfilled';
    setTimeout(function () {
      callbacks.forEach(function (callback) {
        callback(value);
      });
    }, 0);
  }

  fn(resolve);
}
```

`resolve`执行时，就会将状态变为fulfilled，在此之后状态不再变化，调用`then`注册的新回调会立即执行。

至于异步操作失败时的错误处理`reject`会在后面继续讨论。

### 串行Promise

串行Promise也是promise一个非常有用的特性之一。

如下面的例4，如果在`then`函数中再返回一个promise，那么该如何解决？

```javascript
getUserId()
    .then(getUserMobileById)
    .then(function (mobile) {
        // do sth with mobile
    });

function getUserMobileById(id) {
    return new Promise(function (resolve) {
        Y.io('/usermobile/' + id, {
            on: {
                success: function (i, o) {
                    resolve(JSON.parse(o).mobile);
                }
            }
           });
    });
}
```
解决这个问题的关键在于，如何衔接当前的promise和后面的promise。

```javascript
function Promise(fn) {
    var state = 'pending',
        value = null,
        callbacks = [];

    this.then = function (onFulfilled) {
        return new Promise(function (resolve) {
            handle({
                onFulfilled: onFulfilled || null,
                resolve: resolve
            });
        });
    };

    function handle(callback) {
        if (state === 'pending') {
            callbacks.push(callback);
            return;
        }
        //如果then中没有传递任何东西
        if(!callback.onFulfilled) {
            callback.resolve(value);
            return;
        }

        var ret = callback.onFulfilled(value);
        callback.resolve(ret);
    }

    
    function resolve(newValue) {
        if (newValue && (typeof newValue === 'object' || typeof newValue === 'function')) {
            var then = newValue.then;
            if (typeof then === 'function') {
                then.call(newValue, resolve);
                return;
            }
        }
        state = 'fulfilled';
        value = newValue;
        setTimeout(function () {
            callbacks.forEach(function (callback) {
                handle(callback);
            });
        }, 0);
    }

    fn(resolve);
}
```
以例4为例，上面代码的执行步骤如下：

1. `getUserId` 生成的 promise （简称 `getUserId` promise）异步操作成功，执行其内部方法 `resolve`，传入的参数正是异步操作的结果 `userid`；
2. 调用 `handle` 方法处理 `callbacks` 队列中的回调：`getUserMobileById` 方法，生成新的 promise（简称 `getUserMobileById` promise）；
3. 执行之前由 `getUserId` promise 的 `then` 方法生成的 bridge promise 的 `resolve` 方法，传入参数为 `getUserMobileById` promise。这种情况下，会将该 `resolve` 方法传入 `getUserMobileById` promise 的 `then` 方法中，并直接返回；
4. 在 `getUserMobileById` promise 异步操作成功时，执行其 `callbacks` 中的回调：`getUserId` bridge promise 的 `resolve` 方法；
5. 最后，执行 `getUserId` bridge promise 的后邻 promise 的 `callbacks` 中的回调

可以用一张图来描述这个过程：

{% asset_img promise-series-flow.png %}

### 失败处理

之前遗留的rejected状态处理在这里补上，异步操作失败时，状态将由pending变为rejected，并执行注册好的是失败回调：

```javascript
//例5
function getUserId() {
  return new Promise(function(resolve) {
    //异步请求
    http.get(url, function(error, results) {
      if (error) {
        reject(error);
      }
      resolve(results.id)
    })
  })
}

getUserId().then(function(id) {
  //一些处理
}, function(error) {
  console.log(error)
})
```

有了之前的基础，支持错误处理变得很容易，在注册回调、处理了状态变更上都要加入新的逻辑：

```javascript
function Promise(fn) {
    var state = 'pending',
        value = null,
        callbacks = [];

    this.then = function (onFulfilled, onRejected) {
        return new Promise(function (resolve, reject) {
            handle({
                onFulfilled: onFulfilled || null,
                onRejected: onRejected || null,
                resolve: resolve,
                reject: reject
            });
        });
    };

    function handle(callback) {
        if (state === 'pending') {
            callbacks.push(callback);
            return;
        }

        var cb = state === 'fulfilled' ? callback.onFulfilled : callback.onRejected,
            ret;
        if (cb === null) {
            cb = state === 'fulfilled' ? callback.resolve : callback.reject;
            cb(value);
            return;
        }
        ret = cb(value);
        callback.resolve(ret);
    }

    function resolve(newValue) {
        if (newValue && (typeof newValue === 'object' || typeof newValue === 'function')) {
            var then = newValue.then;
            if (typeof then === 'function') {
                then.call(newValue, resolve, reject);
                return;
            }
        }
        state = 'fulfilled';
        value = newValue;
        final();
    }

    function reject(reason) {
        state = 'rejected';
        value = reason;
        final();
    }

    function final() {
        setTimeout(function () {
            callbacks.forEach(function (callback) {
                handle(callback);
            });
        }, 0);
    }

    fn(resolve, reject);
}
```

增加了新的`reject`方法，供异步操作失败时调用，同时增加`final`方法。

错误冒泡是上述代码已经支持，且非常实用的一个特性。在 `handle` 中发现没有指定异步操作失败的回调时，会直接将 bridge promise 设为 rejected 状态，如此达成执行后续失败回调的效果。这有利于简化串行 Promise 的失败处理成本，因为一组异步操作往往会对应一个实际功能，失败处理方法通常是一致的：

```javascript
// 例6

getUserId()
    .then(getUserMobileById)
    .then(function (mobile) {
        // 一些处理
    }, function (error) {
        // getUserId或者getUerMobileById时出现的错误
        console.log(error);
    });
```

### 异常处理

如果在执行成功回调、失败回调时代码出错怎么办？对于这类异常，可以使用 `try-catch` 捕获错误，并将 bridge promise 设为 rejected 状态。`handle` 方法改造如下：

```
function handle(callback) {
    if (state === 'pending') {
        callbacks.push(callback);
        return;
    }

    var cb = state === 'fulfilled' ? callback.onFulfilled : callback.onRejected,
        ret;
    if (cb === null) {
        cb = state === 'fulfilled' ? callback.resolve : callback.reject;
        cb(value);
        return;
    }
    try {
        ret = cb(value);
        callback.resolve(ret);
    } catch (e) {
        callback.reject(e);
    } 
}
```

如果在异步操作中，多次执行 `resolve` 或者 `reject` 会重复处理后续回调，可以通过内置一个标志位解决。

### 总结

Promise作为一种异步操作的解决方案，刚开始使用的时候总是很难理解其中的状态变化和运行机制。本文从最简单的基础实现开始，逐步添加内置状态、串行、失败/异常处理等特性，最终达到了一个简单Promise实现。