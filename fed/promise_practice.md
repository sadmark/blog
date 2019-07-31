# 关于`Promise`的一些练习

前端学习的过程中，知识的获取过于琐碎，通常是这里看一篇博客，那边又看一篇，如此反复，总觉得不够连贯。尝试着对`Promise`的知识点做一些练习，以巩固基础。

## `micro task`

不管什么时候，都能在`javascript`相关的知识点中看到`event loop`，也总是能知道`Promise`是属于`micro task`。

当我们实例化的时候，`new Promise(fn).then(cb)`中的`fn`是同步执行的。之后的`then`调用链中的回调函数才会归属于`micro task`，所以才会有一些充满陷阱的面试题。

```javascript
console.log('script start');
setTimeout(() => console.log('timeout'), 0);
new Promise((resolve) => {
  console.log('new Promise');
  resolve();
}).then(() => console.log('then'));
console.log('script end');

// script start -> new Promise -> script end -> then -> timeout
```

## 如何实现一个`Promise`

由`Promise A+`规范可以知道，一个`Promise`有3种状态，分别是`pending | fulfilled | rejected`。默认的初始状态为`pending`，当状态变更以后，不可逆。同时`Promise`有一个`then`方法，返回一个新的`Promise`。

以下便是一个简易版的实现。

```javascript
const PENDING = 'pending';
const RESOLVED = 'fulfilled';
const REJECTED = 'rejected';

class Promise {
  constructor(fn) {
    this._status = PENDING; // pending | fulfilled | rejected
    this._result = null;
    this._resolvedCb = null;
    this._rejectedCb = null;

    fn(this._resolve.bind(this), this._reject.bind(this));
  }

  _resolve(value) {
    if (this._status === PENDING) {
      this._status = RESOLVED;
      this._result = value;
      if (this._resolvedCb) this._resolvedCb();
    }
  }

  _reject(error) {
    if (this._status === PENDING) {
      this._status = REJECTED;
      this._result = error;
      if (this._rejectedCb) this._rejectedCb();
    }
  }

  _run(result, resolve, reject) {
    try {
      resolve(result);
    } catch (error) {
      reject(result);
    }
  }

  then(resolvedCb, rejectedCb) {
    return new Promise(resolve => {
      if (this._status === RESOLVED) {
        resolve(resolvedCb(this._result));
      } else if (this._status === REJECTED) {
        // 当一个promise error已经被rejectedCb处理后
        // 返回的应该是一个 resolved promise
        resolve(rejectedCb(this._result));
      } else {
        this._resolvedCb = () => resolve(resolvedCb(this._result));
        this._rejectedCb = () => resolve(rejectedCb(this._result));
      }
    });
  }
}
```

以上代码仅实现了一个非常简陋的`Promise`，并没有`catch`、`all`、`race`、`try`等实现，并且没有判断`then`中`resolvedCb`和`rejectedCb`的返回值是否是一个`Promise`对象。之后可以尝试再进行进一步的扩展。

## `Promise`进行流程控制

网上有非常多的关于流程控制的问题，需要用到`Promise`来解决。举个🌰：

```text
一共有10个请求，同时只能发起3个，当其中一个完成后马上进行下一个请求，直至10个请求全部结束。
```

平时`Promise.race`用的机会不是很多，这题的关键其实就是`Promise.race`。

首先，定义一个`ajax`函数来模拟数据请求，并且生成10个`URL`。

```javascript
function ajax(url) {
  const ms = parseInt(Math.random() * 5000, 10);
  return new Promise(resolve => {
    setTimeout(() => {
      console.log(`请求URL: ${url} 完成，耗时：${ms} 毫秒`);
      resolve();
    }, ms);
  });
}

const urls = [...new Array(10).keys()].map(item => `/fake-path/${item + 1}`);
```

接下来就来完成后面的逻辑吧。

```javascript
doFetch(urls, ajax).then(() => console.log('done'));

function doFetch(urls, ajax, limit = 3) {
  const arr = urls.slice(0, limit) // 先取出3个直接执行
                  .map((url, index) => ajax(url).then(() => index)); // 返回index方便后面替换已经完成的promise
  return urls
    .slice(limit) // 取出剩下的做reduce操作
    .reduce((promise, nextUrl) => {
      // 不断操纵promise，直至所有url被请求完
      // 如果arr中哪个ajax先完成了，就替换掉，换成新的ajax请求
      return promise
        .then(() => Promise.race(arr))
        // race完成后会得到map时的index
        .then(index => {
          arr[index] = ajax(nextUrl).then(() => index);
        });
    }, Promise.resolve())
    .then(() => Promise.all(arr)); // 全部reduce完之后等arr数组中的ajax执行完成
}
```

## 参考

[Promise 对象](http://es6.ruanyifeng.com/#docs/promise)
