---
title: Promise A+规范-译文
date: 2021-03-18 14:05:04
tags: promise
---

# Promises/A+

**一个开源的、健全的、可操作的JavaScript Promises规范，由开发者制定，为其他开发者提供参考**

`Promise`代表一个异步操作的最终结果。与之进行交互的主要方法是通过它的`then`方法，该方法注册回调函数接受`Promise`的最终结果或`Promise`未完成的原因。

本规范详细说明了`then`方法的执行过程，所有遵循Promise/A+实现的promise均可以本规范作为参照。因此本规范是十分稳定的。虽然Promises/A+组织有时会通过向后兼容的小改动来修订这份规范，用于处理一些新发现的边界情况，但是只有经过仔细的思考、讨论与测试后，我们才会进行大规模不向下兼容的更新。

从历史上看，Promises/A+把较早的Promise/A提案的建议明确为行为标准，扩展原有约定俗成的行为，并忽略了未特指的情况与有问题的情况。

最后，Promise/A+规范的核心不涉及如何去创建、解决或拒绝Promise,取而代之选择聚焦于如何提供一个可操作的`then`方法。未来在相关规范的工作中可能会涉及这些话题。

## 1. 术语

### 1.1 promise

promise 是定义了行为符合本规范的`then`方法的对象或者函数。

### 1.2 thenable

thenable 表示定义了`then`方法的对象或函数。

### 1.3 value

value 指任何JavaScript合法值（包括undefined，thenable和promise）。

### 1.4 exception

exception 是使用`throw`语句抛出的异常值。

### 1.5 reason

reason 表示一个promise拒绝的原因。

## 2.要求

### 2.1 Promise的状态

一个promise的当前状态必须是以下三种中的一种：pending(等待态) ，fulfilled（完成态） 或 rejected（拒绝态）

2.1.1 当promise状态为pending时：

​        2.1.1.1.  可能转变为fulfilled或rejected。

2.1.2 当promise状态为fulfilled时：

​         2.1.2.1 不会转变为其他状态

​         2.1.2.2 必须有一个确定的值，不会再改变

2.1.3 当promise状态为rejected时，

​         2.1.3.1 不会再变为其他状态

​         2.1.3.2 必须返回确定的`reason`，不会再改变

这里的不可变指的是恒等（即可用 `===` 判断相等），而不是意味着更深层次的不可变（**注：**指当 value 或 reason 不是基本值时，只要求其引用地址相等，但属性值可被更改）。

### 2.2 then方法

一个promise必须提供一个then方法来获取当前值，终值以及拒绝原因。

promise的then方法接受两个参数:

```js
promise.then(onFulfilled, onRejected)
```

2.2.1 参数可选

`onFulfilled`和`onRejected`都是可选的参数：

​        2.2.1.1 如果`onFulfilled`不是一个函数，则会被忽略                       

​        2.2.1.1 如果``onRejected``不是一个函数，则会被忽略

2.2.2  onFulfilled

`onFulfilled`是函数：

​       2.2.2.1 在promise状态变为fulfilled时被调用，promise的返回值作为其第一个入参

​       2.2.2.2 在promise的状态变为fulfilled前不可被调用

​       2.2.2.3 仅可以被回调一次

2.2.3  onRejected

``onRejected``是函数：

​        2.2.3.1 在promise状态变为rejected时被调用，promise的拒绝reason作为其第一个入参

​       2.2.3.2 在promise的状态变为rejected前不可被调用

​       2.2.3.3 仅可以被回调一次

2.2.4 调用时机

`onFulfilled` 和 `onRejected` 只有在[执行环境](http://es5.github.io/#x10.3)堆栈仅包含"platform code"时才可被调用[^3.1] 

2.2.5  调用要求

`onFulfilled` 与 `onRejected` 必须以函数形式被回调（即没有this）[^3.2]

2.2.6 多次调用

then方法可以在同一promise中多次被调用

​        2.2.6.1 如果promise的状态为fulfilled，每个`onFulfilled` 都必须按照他们的注册顺序依次回调。

​       2.2.6.2 如果promise的状态为rejected，每个`onRejected` 都必须按照他们的注册顺序依次回调。

2.2.7 then返回

then方法必须返回一个promise对象[^3.3]

```js
 promise2 = promise1.then(onFulfilled, onRejected);
```

​          2.2.7.1 如果`onFulfilled` 或`onRejected` 返回值x，执行Promise解决程序`[[Resolve]](promise2,x)`。

​         2.2.7.2 如果`onFulfilled` 或`onRejected` 抛出异常 `e`,promise必须拒绝执行，并以` e `为reason。

​         2.2.7.3 如果 `onFulfilled` 不是一个函数并且promise1的状态为fulfilled，promise2必须成功执行，并返回与promise1相同的值。

​       2.2.7.4 如果 `onRejected` 不是一个函数并且promise2的状态为rejected，promise2必须拒绝执行，并返回与promise1相同的reason。

### 2.3 Promise解决过程

promise解决过程是一个抽象的操作，以一个promise和值作为输入，表示为`[[Resolve]](promise,x)`。如果`x`具有then方法且看上去像一个promise，解决程序则会使promise接受x的状态。否则，用 `x` 的值来执行 `promise`。

这种thenable的特性使得promise的实现更具有通用性，只要其暴露一个符合Promise/A+规范的then方法。同时也使得遵循Promise/A+规范的实现与不太规范的实现能兼容。

运行 `[[Resolve]](promise, x)` 需遵循以下步骤：

2.3.1 x与promise相同

如果Promise与x指向同一对象，以TypeError为理由拒绝执行

2.3.2 x是promise

如果x 是promise，则使 `promise` 接受 `x` 的状态 [^3.4]

​       2.3.2.1 如果x处于pending状态，promise必须保持pending状态直到x状态变为fulfilled或rejected。

​       2.3.2.2 如果x处于fulfilled状态，fulfilled状态的promise必须有同样的值

​       2.3.2.3 如果x处于rejected状态，rejected状态的promise必须有相同的reason

2.3.3 x是对象或者函数

如果x是一个对象或者函数

​      2.3.3.1 将`x.then` 赋值给 `then`[^3.5]

​      2.3.3.2  如果取`x.then`的值时抛出异常`e`,以`e`为原因拒绝promise

​      2.3.3.3  如果`then`是一个函数，以`x`作为`this`调用，`then`方法的参数为两个回调函数，第一个参数是`resolvePromise`，第二个参数是`rejectPromise`：

​            2.3.3.3.1 当使用`y`值调用`resolvePromise`，运行`[[Resolve]](promise, y)`

​            2.3.3.3.2  当使用`r`为原因调用`rejectPromise`，以`r`为拒因拒绝promise

​            2.3.3.3.3  如果同时调用`resolvePromise`和`rejectPromise`或对同一个参数进行了多次调用，则优先第一个调用优先，其他任何调用都将被忽略。

​            2.3.3.3.4 如果`then`方法抛出了异常`e`

​                    2.3.3.3.4.1 如果已调用`resolvePromise`或`rejectPromise`，则忽略。

​                    2.3.3.3.4.2 否则，以`e`为原因拒绝promise

​      2.3.3.4 如果`then`方法不是函数，则以`x`为参数执行promise

2.3.4  `x`不是函数或对象，则以`x`为参数执行promise

如果promise被一个循环的thenable链中的thenable对象执行，而因为`[[Resolve]](promise, y)`的递归属性，使得`[[Resolve]](promise, y)`被再次调用，根据上述算法产生无限递归的情况。算法鼓励但不强制要求去检测这种递归情况是否存在，如果检测到这种情况存在，则以TypeError为原因拒绝执行[^3.6]。

## 3 注释

[^3.1]: "plantform code"指的是引擎、环境、promise实施代码。实际中确保`onFulfilled`和`onRejected`异步执行，在then方法被调用的那一轮事件调用结束后使用新的栈。可以使用“macro-task”机制例如`setTimeout`或`setImmediate`或使用“micro-task”机制例如`MutationObserver`或`process.nextTick`执行。由于 promise 的实施代码本身就是"plantform code"（**注：**即都是 JavaScript），故代码自身在处理在处理程序时可能已经包含一个任务调度队列。
[^3.2]: 在严格模式下，`this`的值为`undefined`;在非严格模式下，`this`为全局对象。
[^3.3]: 代码实现在满足所有要求的情况下可以允许 `promise2 === promise1` 。每个实现都要说明其是否允许以及在何种条件下允许 `promise2 === promise1` 。
[^3.4]: 通常，若 `x` 符合当前实现，我们才认为它是真正的 *promise* 。这一规则允许那些特例实现采用符合已知要求的 Promises 状态。
[^3.5]: 该过程我们先存储了`x.then`的引用，然后测试这个引用，再调用该引用，避免了对x.then属性的多次访问。这种预防措施对于保护访问者属性的一致性非常重要，因为访问者属性值在两次检索期间可能发生变化。
[^3.6]: 实现不应该对thenable链进行深度的限制，并且假设超出深度限制的就是无限递归。只有真正的无限递归会抛出`TypeError`;如果是一条无限长的链上 *thenable* 均不相同，那么递归下去永远是正确的行为。

