---
title: 深入JS之Promise
date: 2020-06-27
tags:
  - JavaScript
  - Promise
---

本篇博客主要是对 Javcscript 异步编程 Promise 的学习理解
<!-- more -->
## 基本概念
**MDN 对 Promise 的定义：**
> Promise 是一个对象，它代表了一个异步操作的最终完成或者失败。

Promise 解决的主要是异步编码风格的问题，即解决回调地狱的问题。在开发一些稍复杂点项目时，你会发现，如果嵌套了太多的回调函数就很容易使得自己陷入了回调地狱，不能自拔。你可以参考下面这段让人凌乱的代码：

```
XFetch(makeRequest('https://xxx'),
      function resolve(response) {
          console.log(response)
          XFetch(makeRequest('https://xxx'),
              function resolve(response) {
                  console.log(response)
                  XFetch(makeRequest('https://xxx')
                      function resolve(response) {
                          console.log(response)
                      }, function reject(e) {
                          console.log(e)
                      })
              }, function reject(e) {
                  console.log(e)
              })
      }, function reject(e) {
          console.log(e)
      })
```

这段代码是先请求第一个接口，如果请求成功的话，那么再请求第二个接口，如果再次请求成功的话，就继续请求最后一个接口。也就是说这段代码用了三层嵌套请求，就已经让代码变得混乱不堪，当上述需求中出现第四个请求（甚至更多）仍然依赖上一个请求的时候，代码就会变成一场灾难。

这段代码之所以看上去很乱，归结其原因有两点：
- **第一是嵌套调用**，下面的任务依赖上个任务的请求结果，并**在上个任务的回调函数内部执行新的业务逻辑**，这样当嵌套层次多了之后，代码的可读性就变得非常差了。
- **第二是任务的不确定性**，执行每个任务都有两种可能的结果（成功或者失败），所以体现在代码中就需要对每个任务的执行结果做两次判断，这种对每个任务都要进行一次额外的错误处理的方式，明显增加了代码的混乱程度。

这时，我们可能会希望：

- [x] **第一是消灭嵌套调用；**
- [x] **第二是合并多个任务的错误处理**。
    
**Promise 主要通过下面两步解决嵌套回调问题的。**

1. **首先，Promise 实现了回调函数的延时绑定**。回调函数的延时绑定在代码上体现就是先创建 Promise 对象 promise1，通过 Promise 的构造函数 executor 来执行业务逻辑；创建好 Promise 对象 promise1 之后，再使用 promise1.then 来设置回调函数。

2. **其次，需要将回调函数 onResolve 的返回值穿透到最外层**。因为我们会根据 onResolve 函数的传入值来决定创建什么类型的 Promise 任务，创建好的 Promise 对象需要返回到最外层，这样就可以摆脱嵌套循环了。代码如下：

```
let promise1 = new Promise((resolve, reject) => {
    console.log(1)
    resolve('success')
})

// then 方法延迟绑定 返回值穿透赋给 p2
let p2 = promise1.then(res => {
    console.log(res)
    return 2
})
setTimeout(() => {
   console.log(p2) 
})
// 输出 1 ,success, 2
```

Promise 通过回调函数延迟绑定和回调函数返回值穿透的技术，解决了循环嵌套。
接下来我们再来看看 Promise 是怎么处理异常的：

```

function executor(resolve, reject) {
    let rand = Math.random();
    console.log(1)
    console.log(rand)
    if (rand > 0.5)
        resolve()
    else
        reject()
}
var p0 = new Promise(executor);

var p1 = p0.then((value) => {
    console.log("succeed-1")
    return new Promise(executor)
})
p1.catch((error) => {
    console.log("error")
})
console.log(2)
```
上面 p0, p1 无论哪个对象报错都可以在最后 catch 里面捕获到。
之所以可以使用最后一个对象来捕获所有异常，是因为 Promise 对象的错误具有“冒泡”性质，会一直向后传递，直到被 onReject 函数处理或 catch 语句捕获为止。具备了这样“冒泡”的特性后，就不需要在每个 Promise 对象中单独捕获异常了。

## 原理剖析
上面我们了解 Promise 的实现主要是通过延迟回调、返回值穿透以及错误的‘冒泡’捕获。

接下来，我们通过源码实现来更深层次的理解这些技术实现，深入理解 Promise。

### 极简Promise雏形
```
function Promise(executor) {
  this.value = null   // 存储 resolved 的值
  this.reason = null // 存储 rejected 的值
  this.onFulfilledArray = [] // 可能同时有多个回调，用数组来存储
  this.onRejectedArray = [] //可能同时有多个回调，用数组来存储

  const resolve = value => {
    this.value = value
    this.onFulfilledArray.forEach(func => {
      func(value)
    })
  }

  const reject = reason => {
    this.reason = reason
    this.onRejectedArray.forEach(func => {
      func(reason)
    })
  }
  
   executor(resolve, reject)
}

Promise.prototype.then = function(onfulfilled, onrejected) {
    this.onFulfilledArray.push(onfulfilled)
    this.onRejectedArray.push(onrejected)
}
```
上述代码比较简单，大概的逻辑是这样的：
1. 调用 then 方法，将想要在 Promise 异步操作成功时执行的回调放入 onFulfilledArray 队列里，失败时执行的回调放在 onRejectedArray 队列里。
2. 创建 Promise 实例时传入的函数会被赋予函数类型的参数，即 resolve 和 reject，它们接收一个参数，代表异步操作返回的结果，当一步操作执行成功后，用户会调用 resolve 方法。失败时，会执行 reject 方法。这时候其实真正执行的操作是将队列中的回调一一执行。

### 加入延迟绑定机制
细心的小伙伴可能会发现，上述的代码还存在一些问题：在 then 回调之前， resolve 和 reject 函数就会执行。因此，需要将 resolve 和 reject 的执行，放到任务队列中。这里姑且先放到 setTimeout 里，保证异步执行（这样的做法并不严谨，为了保证 Promise 属于 microtasks，很多 Promise 的实现库用了 MutationObserver 来模仿 nextTick）

```
 const resolve = value => {
    // setTimeout 模拟异步执行
    setTimeout(() => { 
      this.value = value
      this.onFulfilledArray.forEach(func => {
        func(value)
      })
    })
  }

  const reject = reason => {
    // setTimeout 模拟异步执行
    setTimeout(() => { 
      this.reason = reason
      this.onRejectedArray.forEach(func => {
        func(reason)
      })
    })
  }
```
### 加入状态
Promises/A+ 规范中的 Promise States 中明确规定了，pending 可以转化为 fulfilled 或 rejected 并且只能转化一次，也就是说如果 pending 转化到 fulfilled 状态，那么就不能再转化到 rejected。并且 fulfilled 和 rejected 状态只能由 pending 转化而来，两者之间不能互相转换。一图胜千言：

![](https://user-gold-cdn.xitu.io/2020/6/8/17292885fc45811c?w=353&h=260&f=png&s=4035)
改进后的代码：

```
function Promise(executor) {
  this.status = 'pending'
  this.value = null
  this.reason = null
  this.onFulfilledArray = []
  this.onRejectedArray = []

  const resolve = value => {
    // setTimeout 模拟异步执行
    setTimeout(() => { 
      if (this.status === 'pending') {
        this.value = value
        this.status = 'fulfilled'

        this.onFulfilledArray.forEach(func => {
          func(value)
        })
      }
    })
  }

  const reject = reason => {
    // setTimeout 模拟异步执行
    setTimeout(() => { 
      if (this.status === 'pending') {
        this.reason = reason
        this.status = 'rejected'
        
        this.onRejectedArray.forEach(func => {
          func(reason)
        })
      }
    })
  }
  executor(resolve, reject)
}

Promise.prototype.then = function(onfulfilled, onrejected) {
  if (this.status === 'fulfilled') {
    onfulfilled(this.value)
  }
  if (this.status === 'rejected') {
    onrejected(this.reason)
  }
  if (this.status === 'pending') {
    this.onFulfilledArray.push(onfulfilled)
    this.onRejectedArray.push(onrejected)
  }
}
```
上述代码的思路是这样的：resolve 和 reject 执行时，会将状态设置为 fulfilled 和 rejected， 在此之后调用 then 添加的新回调，都会立即执行。

### 链式Promise
先从下面例题说起：

```
const promise = new Promise((resolve, reject) => {
  setTimeout(() => {
      resolve('lucas')
  }, 2000)
})

promise.then(data => {
  console.log(data)
  return new Promise((resolve, reject) => {
    setTimeout(() => {
        resolve(`${data} next then`)
    }, 4000)
  })
})
.then(data => {
  console.log(data)
})
```
将在 2 秒后输出：lucas，紧接着再过 4 秒后（第 6 秒）输出：lucas next then。

这种场景相信用 过promise 的人都知道会有很多，那么类似这种就是所谓的链式 Promise。

一个 Promise 实例的 then 方法体 onfulfilled 函数和 onrejected 函数中，是支持再次返回一个 Promise 实例的，也支持返回一个非 Promise 实例的普通值；并且返回的这个 Promise 实例或者这个非 Promise 实例的普通值将会传给下一个 then 方法 onfulfilled 函数或者 onrejected 函数中，这样就支持链式调用了。

对 then 方法进行代码改造后：

<details>

<summary> 点击展开完整代码 </summary>

    // then 方法中 变量 result 既可以是一个普通值，也可以是一个 Promise 实例，为此我们抽象出 resolvePromise 方法进行统一处理 
    
    const resolvePromise = (promise2, result, resolve, reject) => {
      // 当 result 和 promise2 相等时，也就是说 onfulfilled 返回 promise2 时，进行 reject
      if (result === promise2) {
        return reject(new TypeError('error due to circular reference'))
      }
    
      // 是否已经执行过 onfulfilled 或者 onrejected
      let consumed = false
      let thenable
    
      if (result instanceof Promise) {
        if (result.status === 'pending') {
          result.then(function(data) {
            resolvePromise(promise2, data, resolve, reject)
          }, reject)
        } else {
          result.then(resolve, reject)
        }
        return
      }
    
      let isComplexResult = target => (typeof target === 'function' || typeof target === 'object') && (target !== null)
      // 如果返回的是疑似 Promise 类型
      if (isComplexResult(result)) {
        try {
          thenable = result.then
          // 如果返回的是 Promise 类型，具有 then 方法
          if (typeof thenable === 'function') {
            thenable.call(result, function(data) {
              if (consumed) {
                return
              }
              consumed = true
    
              return resolvePromise(promise2, data, resolve, reject)
            }, function(error) {
              if (consumed) {
                return
              }
              consumed = true
    
              return reject(error)
            })
          }
          else {
            return resolve(result)
          }
    
        } catch(e) {
          if (consumed) {
            return
          }
          consumed = true
          return reject(e)
        }
      }
      else {
        return resolve(result)
      }
    }
    
    Promise.prototype.then = function(onfulfilled, onrejected) {
      onfulfilled = typeof onfulfilled === 'function' ? onfulfilled : data => data
      onrejected = typeof onrejected === 'function' ? onrejected : error => {throw error}
    
      // promise2 将作为 then 方法的返回值
      let promise2
    
      if (this.status === 'fulfilled') {
        return promise2 = new Promise((resolve, reject) => {
          setTimeout(() => {
            try {
              // 这个新的 promise2 resolved 的值为 onfulfilled 的执行结果
              let result = onfulfilled(this.value)
              resolvePromise(promise2, result, resolve, reject)
            }
            catch(e) {
              reject(e)
            }
          })
        })
      }
      if (this.status === 'rejected') {
        return promise2 = new Promise((resolve, reject) => {
          setTimeout(() => {
            try {
              // 这个新的 promise2 reject 的值为 onrejected 的执行结果
             let result = onrejected(this.reason)
             resolvePromise(promise2, result, resolve, reject)
            }
            catch(e) {
              reject(e)
            }
          })
        })
      }
      if (this.status === 'pending') {
        return promise2 = new Promise((resolve, reject) => {
          this.onFulfilledArray.push(value => {
            try {
              let result = onfulfilled(value)
              resolvePromise(promise2, result, resolve, reject)
            }
            catch(e) {
              return reject(e)
            }
          })
    
          this.onRejectedArray.push(reason => {
            try {
              let result = onrejected(reason)
              resolvePromise(promise2, result, resolve, reject)
            }
            catch(e) {
              return reject(e)
            }
          })      
        })
      }
    }

</details>

### Promise 值穿透
看下面例子：

```
const promise = new Promise((resolve, reject) => {
  setTimeout(() => {
      resolve('lucas')
  }, 2000)
})


promise.then(null)
.then(data => {
  console.log(data)
})
```
这段代码将会在 2 秒后输出：lucas。这就是 Promise 穿透现象：

给 .then() 函数传递非函数值作为其参数时，实际上会被解析成 .then(null)，这时候的表现应该是：上一个 promise 对象的结果进行“穿透”，如果在后面链式调用仍存在第二个 .then() 函数时，将会获取被穿透下来的结果。

值穿透代码实现：

```
Promise.prototype.then = function(onfulfilled = Function.prototype, onrejected = Function.prototype) {
  // 如果 then 方法接收的不是函数，则默认赋予直接返回其值函数，从而实现穿透
  onfulfilled = typeof onfulfilled === 'function' ? onfulfilled : data => data
  onrejected = typeof onrejected === 'function' ? onrejected : error => { throw error }

    // ...
}
```

### 异常处理
Promise.prototype.catch 可以进行异常捕获，它的典型用法：

```
const promise1 = new Promise((resolve, reject) => {
  setTimeout(() => {
      reject('lucas error')
  }, 2000)
})

promise1.then(data => {
  console.log(data)
}).catch(error => {
  console.log(error)
})
```
会在 2 秒后输出：lucas error。

其实在这种场景下，它就相当于：

```
Promise.prototype.catch = function(catchFunc) {
  return this.then(null, catchFunc)
}
```
因为我们知道 .then() 方法的第二个参数也是进行异常捕获的，通过这个特性，我们比较简单地实现了 Promise.prototype.catch。
### Promise.prototype.resolve 实现
请看实例：

```
Promise.resolve('data').then(data => {
  console.log(data)
})
console.log(1)
```
先输出 1 再输出 data。

那么实现 Promise.resolve(value) 也很简单：

```
Promise.resolve = function(value) {
  return new Promise((resolve, reject) => {
    resolve(value)
  })
}
```
顺带实现一个 **Promise.reject(value)**：


```
Promise.reject = function(value) {
  return new Promise((resolve, reject) => {
    reject(value)
  })
}
```
## 总结
最后把完整代码贴出：

<details>

<summary> 点击展开完整代码 </summary>

    function Promise(executor) {
      this.status = 'pending'
      this.value = null
      this.reason = null
      this.onFulfilledArray = []
      this.onRejectedArray = []
    
      const resolve = value => {
        if (value instanceof Promise) {
          return value.then(resolve, reject)
        }
        setTimeout(() => {
          if (this.status === 'pending') {
            this.value = value
            this.status = 'fulfilled'
    
            this.onFulfilledArray.forEach(func => {
              func(value)
            })
          }
        })
      }
    
      const reject = reason => {
        setTimeout(() => {
          if (this.status === 'pending') {
            this.reason = reason
            this.status = 'rejected'
    
            this.onRejectedArray.forEach(func => {
              func(reason)
            })
          }
        })
      }
    
    
        try {
            executor(resolve, reject)
        } catch(e) {
            reject(e)
        }
    }
    
    const resolvePromise = (promise2, result, resolve, reject) => {
      // 当 result 和 promise2 相等时，也就是说 onfulfilled 返回 promise2 时，进行 reject
      if (result === promise2) {
        return reject(new TypeError('error due to circular reference'))
      }
    
      // 是否已经执行过 onfulfilled 或者 onrejected
      let consumed = false
      let thenable
    
      if (result instanceof Promise) {
        if (result.status === 'pending') {
          result.then(function(data) {
            resolvePromise(promise2, data, resolve, reject)
          }, reject)
        } else {
          result.then(resolve, reject)
        }
        return
      }
    
      let isComplexResult = target => (typeof target === 'function' || typeof target === 'object') && (target !== null)
      // 如果返回的是疑似 Promise 类型
      if (isComplexResult(result)) {
        try {
          thenable = result.then
          // 如果返回的是 Promise 类型，具有 then 方法
          if (typeof thenable === 'function') {
            thenable.call(result, function(data) {
              if (consumed) {
                return
              }
              consumed = true
    
              return resolvePromise(promise2, data, resolve, reject)
            }, function(error) {
              if (consumed) {
                return
              }
              consumed = true
    
              return reject(error)
            })
          }
          else {
            return resolve(result)
          }
    
        } catch(e) {
          if (consumed) {
            return
          }
          consumed = true
          return reject(e)
        }
      }
      else {
        return resolve(result)
      }
    }
    
    Promise.prototype.then = function(onfulfilled, onrejected) {
      onfulfilled = typeof onfulfilled === 'function' ? onfulfilled : data => data
      onrejected = typeof onrejected === 'function' ? onrejected : error => {throw error}
    
      // promise2 将作为 then 方法的返回值
      let promise2
    
      if (this.status === 'fulfilled') {
        return promise2 = new Promise((resolve, reject) => {
          setTimeout(() => {
            try {
              // 这个新的 promise2 resolved 的值为 onfulfilled 的执行结果
              let result = onfulfilled(this.value)
              resolvePromise(promise2, result, resolve, reject)
            }
            catch(e) {
              reject(e)
            }
          })
        })
      }
      if (this.status === 'rejected') {
        return promise2 = new Promise((resolve, reject) => {
          setTimeout(() => {
            try {
              // 这个新的 promise2 reject 的值为 onrejected 的执行结果
             let result = onrejected(this.reason)
             resolvePromise(promise2, result, resolve, reject)
            }
            catch(e) {
              reject(e)
            }
          })
        })
      }
      if (this.status === 'pending') {
        return promise2 = new Promise((resolve, reject) => {
          this.onFulfilledArray.push(value => {
            try {
              let result = onfulfilled(value)
              resolvePromise(promise2, result, resolve, reject)
            }
            catch(e) {
              return reject(e)
            }
          })
    
          this.onRejectedArray.push(reason => {
            try {
              let result = onrejected(reason)
              resolvePromise(promise2, result, resolve, reject)
            }
            catch(e) {
              return reject(e)
            }
          })      
        })
      }
    }
    
    Promise.prototype.catch = function(catchFunc) {
      return this.then(null, catchFunc)
    }
    
    Promise.resolve = function(value) {
      return new Promise((resolve, reject) => {
        resolve(value)
      })
    }
    
    Promise.reject = function(value) {
      return new Promise((resolve, reject) => {
        reject(value)
      })
    }

</details>

## 参考文献
- [x] [30分钟，让你彻底明白Promise原理](https://mengera88.github.io/2017/05/18/Promise%E5%8E%9F%E7%90%86%E8%A7%A3%E6%9E%90/)
- [x] [浏览器工作原理与实践](https://time.geekbang.org/column/article/136895)
- [x] [前端开发核心知识进阶](https://gitbook.cn/gitchat/column/5c91c813968b1d64b1e08fde/topic/5cbbe946bbbba80861a35bfc)