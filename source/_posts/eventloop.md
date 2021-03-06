---
title: eventloop
date: 2021-02-23 17:35:08
tags:
---

# 浅谈JavaScript中的Event Loop

## 概述

众所周知，JavaScript是一门单线程的语言。准确的来说，是它的主线程是单线程，设计成单线程主要是因为JavaScript一开始是设计出来操作浏览器的Dom元素的。单线程可以规避多线程中复杂的状态同步，减轻了浏览器的渲染难度，不会出现多个线程操作同一Dom元素使得浏览器混乱。单线程也意味着，在JavaScript中同一时间只能做一件事情。但这带来一个问题，只有一个执行序列，JavaScript运行时遇到异步操作，会被阻塞。而JavaScript不能阻塞在那里等着异步操作完成，否则整个运行时都被阻塞。

那么JavaScript一门单线程的语言是怎么实现异步操作的？JavaScript运行时又是怎么感知并调用对应的回调函数呢？解决方案是依赖宿主环境的event loop机制，event loop在ES6 中没有提及，主要是靠浏览器按照HTML规范实现的，以及NodeJs实现的event loop, NodeJs与HTML规范中的event loop并不相同，今天仅谈浏览器中的event loop。

这是一个异步调用的小例子，右侧是输出的运行结果。接下来对于Event Loop的介绍能解释为什么会有这样的输出结果。 

## JavaScript Runtime                             

 在介绍event loop之前，我们先来介绍 JavaScript的运行时（Runtime）。从图中可以看到，JS引擎中有两大部分，一是内存堆，就是内存分配发生的地方，我们声明的变量和函数都实体存储在这里；二是函数调用栈，也就是常说的执行上下文栈，当遇到全局上下文或者函数上下文等可执行代码时，会将其推入执行栈中，JS引擎不断的执行栈顶的执行上下文，直到整个执行栈为控。JS引擎只有这唯一的解析执行JS的执行栈，这就是它的主线程。其实这样看来，JS引擎只会“傻傻”的执行栈顶函数，跟异步调用没什么关系。实际上是运行环境（runtime）去管理在什么时候压入什么函数来执行。从另一个侧面也能反映执行栈跟异步调用没什么关系，执行上下文栈是写在[ECMA-262](https://tc39.github.io/ecma262/#execution-context-stack) 规范中，需要遵守它的是浏览器的JavaScript引擎，比如 V8、Quantum等。Event Loop的是写在 [HTML规范](https://www.w3.org/TR/html5/webappapis.html#event-loops)中，需要遵守它的是各个浏览器，比如 Chrome、Firefox等。那么执行栈没办法处理的异步任务，会扔给浏览器里的谁来处理呢？——Web APIs。Web APIs是浏览器提供的一些额外的线程，通常由C++ 实现的，来处理非同步的Dom事件，https 请求，setTimeOut等定时器任务。既然Web APIs可以处理这些事情，为什么不让Web APIs直接把回调函数放回执行栈中呢？那这样会产生另一个问题，回调函数会随机在整个程序中出现，因为Web APIs处理完了就扔回来了。所以才又有了任务队列（这里图中写成Callback queue回调队列即放置回调函数的队列）和event loop，Web APIs在执行完毕后将回调函数推入对应的任务队列，由event loop 充当调度者的角色，去检查调用栈与任务队列。当调用栈为空了，就把一个任务推进入执行，这个执行完了，将下一个推进去。所以整体运行时过程可以总结如下：JavaScript顺序执行，遇到同步任务在主线程中，按顺序从前往后执行，如 script；遇到异步任务这种耗时复杂的操作，主线程遇到会先挂起，丢给 Web APIs找到对应线程去处理，Event loop等主线程空闲，再把已经执行完的回调拿回主线程执行。所以JS引擎的执行栈只负责取消息，event loop只负责产生消息。

这些Web APIs，task queue都是浏览器内核维护的。浏览器维护了大致五类线程，GUI渲染线程，负责浏览器的重绘、渲染等，与JS引擎线程互斥；JS 引擎线程遇到不能处理的异步任务，交给定时器线程，HTTP请求线程等处理，回调结果由事件触发线程进行接管，同时事件触发线程也负责将用户交互事件（如鼠标点击、键盘输入）放入任务队列。

## What is  event loop

浏览器实现的event loop这个调度的大脑，到底是什么？在HTML 规范中，是这样描述event loop的：

*To coordinate events, user interaction, scripts, rendering, networking, and so forth, user agents must use event loops as described in this section. Each agent has an associated event loop, which is unique to that agent.*

*（为了协调事件，用户交互，脚本，界面渲染，网络等等，宿主环境必须使用下一节描述的 event loop。每个浏览器界面都有对应的 event loop）*

可以看到上面写的是 event loops ，我们每个界面并不止一个 event loop，而是有多个 event loop，不同的 event loop管理的方向是完全不同的，这里规范中也有相应的介绍：

*The event loop of a similar-origin window agent is known as a window event loop. The event loop of a dedicated worker agent, shared worker agent, or service worker agent is known as a worker event loop. And the event loop of a worklet agent is known as a worklet event loop.*

- window event loop：通常我们讲的都是这个，管理同源页面的event loop；

- worker event loop：涉及到dedicated web worker（专用线程），shared web worker（共享线程），每个 worker 线程都有个与之关联的 event loop；

-  worklet event loop：woklet 是可以访问渲染引擎的线程，我们一般用不到

我们接下来讲的event loop 都默认指代 window event loop，后两者我们应用并不多，worker 可以看作一个不能访问 Dom 的 JavaScript 运行环境，而 worklet 还处于草案阶段，主要应用于需要超高性能场景（worklet 中跑的是机器码而不是 JavaScript）。

### rules

规范中介绍，Event loop 有以下属性：

1、 一个event loop可以有多个任务队列，一个任务队列是任务的集合

*Task queues are sets, not queues, because step one of the event loop processing model grabs the first runnable task from the chosen queue, instead of dequeuing the first task.*

规范中也说明了task，包括：HTML解析，回调、获取资源，操作Dom

2、 一个微任务队列，微任务队列不是任务队列。

*A microtask is a colloquial way of referring to a task that was created via the queue a microtask algorithm.*

但是在看文档时，很多对于task queue 的描述只有一个，不管是网络还是用户交互，或者计时器，都被Web APIs排进一个 task queue。但实际上规范中明确提及有多个 task queue，并举例子来解释为什么存在多个task queue。一个user agent可以有一个处理键盘鼠标事件的 task queue, 还有一个 task queue 来处理所有其他的事件。以 75% 的几率先处理鼠标和键盘的事件，这样既不会彻底不执行其他 task queues 的前提下保证用户界面的响应， 而且不会让来自同一个task queue的事件顺序错乱。task queue,虽然叫任务队列，但是其实是集合，因为event loop模型的第一步是从选择的队列中获取第一个可运行的任务，而不是第一个任务出队。

### task source

Event loop 管理的是task且有多个task queue，那么是如何将任务分配进入队列的？依据就是task source——任务源。任务源就是这个任务是从哪里来的，是从addEventListener来的呢，还是从setTimeout来的。每个来自相同task source并由相同的event loop管理的task 都必须加入同一个task queue，来自不同task source 的 task 可能会排入不同的 task queue中。也就是说task queue与task source是一个对多的关系，一个task queue 中可能排列着来自不同 task sources的 tasks。为什么要这么区分呢？因为任务分贵贱，比如键盘和鼠标事件，就要把它的响应优先级提高，以便尽可能的提高网页浏览的用户体验。但是具体什么task source对应什么 task queue，规范并没有具体说明。但规范对通用task source 进行了介绍：DOM 操作任务源；用户交互任务源，比如鼠标键盘的操作；网络请求任务源；历史浏览任务源，将history.back 等api 排入 task queue。

### 运行过程

现在我们可以具体来看看event loop是怎么执行的。Event loop会在整个页面存在时不停的执行以下步骤：

1. 选取一个有可执行任务的task queue，user agent可以选择任何task队列，如果没有可选的task queue，则跳到下边的microtasks步骤。 

2. 在task queue中选择oldestTask，将其设置为正在运行的task。 

3. Run: 运行被选择的task。 

4. 将Event Loop的current running task变为null。 

5. 从task队列里移除前边运行的task。

6. Microtasks: 执行Microtasks任务检查（也就是执行microtasks队列里的任务） 

7. 更新渲染（Update the rendering）：可以简单理解为浏览器渲染
8. 如果接下来没有要执行的 task/microstask，event loop 将会计算空闲时间。

9. 返回到第一步。

这里会有一点疑惑，前文说到task queue不是队列是集合，为什么还会出现选择oldestTask的情况？个人理解是取可执行任务中oldestTask。那我们再来看看微任务检查点做了什么：

当user agent去执行一个microtask checkpoint，如果microtask checkpoint的flag（标识）为false，user agent必须运行下面的步骤：

1.将microtask checkpoint的flag设为true。

2.Microtask queue handling: 如果event loop的microtask队列为空，直接跳到第八步（Done）。

3.在microtask队列中选择最老的一个任务。

4.将上一步选择的任务设为event loop的currently running task。

5.运行选择的任务。

6.将event loop的currently running task变为null。

7.将前面运行的microtask从microtask队列中删除，然后返回到第二步（Microtask queue handling）。

8.Done: 每一个environment settings object它们的 responsible event loop就是当前的event loop，会给environment settings object发一个 rejected promises 的通知。

9.清理IndexedDB的事务。

10.将microtask checkpoint的flag设为false。

从上述运行过程可以看出，当微任务队列的标记被写为true之后，只要microtask的队列不为空，event loop中的当前任务就会按顺序执行microtask队列中的所有任务。而任务队列是每次仅执行一个任务而微任务microtask队列是独立的一个队列，在event loop执行过程中才进入到任务队列task queue一次执行。

上述这么多步骤，简单来说event loop在一直进行宏任务-微任务-宏任务的过程，在执行完微任务后，可能会进行浏览器的渲染。

### What is  tasks

那么什么是宏任务？在规范中其实并没有提到marcotask的概念，仅有task的描述。在其他文献中找到这样一句解释：*One go-around of the event loop will have exactly one task being processed from the macrotask queue (this queue is simply called the task queue in the WHATWG specification).*

既然每个event loop都必须有一个来自宏任务队列的任务在执行，那微任务又怎么说呢？这就跟之前提到的微任务队列是独立于任务队列有关，整个任务可以理解为，所有的宏任务放在一个宏任务队列（即任务队列），处理完一个宏任务(从sccript开始)，将微任务队列（包含当时所有的微任务）压入任务队列（宏任务队列）并执行，之后再取下一个任务队列（宏任务）中的宏任务。通常被认为是宏任务的有：

•    script（整体代码）

•    setTimeout

•    setInterval

•    setImmediate

•    I/O

•    UI rendering

### What is microtask

那么微任务又是什么？初始不能被放入任务队列的就是微任务，微任务分两类：回调微任务跟复合微任务，通常只会碰到回调微任务。微任务的也没有具体的分类，通常我们把以下任务认为是微任务：

•    process.nextTick（Node.js 环境）

•    Promise（这里指浏览器实现的原生 Promise）

•    Object.observe（已被 MutationObserver 替代）

•    MutationObserver

•    postMessage

我们习惯把宏任务和微任务都理解成 JavaScript 异步执行的一种形式。事实上只有宏任务是异步的，而微任务是对宏任务代码执行顺序的再分配；宏任务执行完后总是会执行完所有微任务，这种意义上微任务是阻塞主线程的，如果你在某个微任务中不断创建新的微任务，毫无疑问界面会出现假死。所有微任务的意义在于执行一些总是想在某个任务完成后再执行的代码。

### summary

现在对宏任务中的具体函数在进行一点补充，setInterval和setTimeout类似，只是前者是循环的执行。对于执行顺序来说，setInterval会每隔指定的时间将注册的函数置入Event Queue，如果前面的任务耗时太久，那么同样需要等待。setTimeout(fn，0)表示当主线程执行栈为空时，立即执行该函数，即实际执行时间会比预计的长。setInterval(fn, ms)我们已经知道不是每过ms秒会执行一次fn，而是每过ms秒，会有fn进入task Queue。一旦setInterval的回调函数fn执行时间超过了延迟时间ms，那么就完全看不出来有时间间隔了。

微任务中经常会碰到的就是Promise，这里简单介绍一点，Promise其实是一个异步容器，构造函数的参数函数是完完全全的同步代码，只有状态改变触发的then、catch回调才是异步代码，   它只关心状态改变后的异步执行，并且承诺给你一个稳定的结果，嵌套Promise,这也有一个小例子：

这是一个四层嵌套的Promise，因为最里层的Promise需要等待几秒才resolve，所以最外层的Promise返回的实例也要等待几秒才会打印日志。也就是说，只有最里层的Promise状态变成fulfilled，最外层的Promise状态才会变成fulfilled。

Async/await是一个Promise语法糖，await 实际上是一个让出线程的标志。每次使用 await, 解释器都创建一个 promise 对象，然后把剩下的 async 函数中的操作放到 then 回调函数中。await后面的表达式会先执行一遍，将await后面的代码加入到microtask中，然后就会跳出整个async函数来执行后面的代码。

现在在介绍了event loop后可以看下最开始的例子的结果是什么顺序执行出来的。

首先，全局代码作为宏任务进入执行栈，setTimeout为宏任务，先加入宏任务队列，Promise为微任务，也加入微任务队列。在第一个宏任务执行完毕后，近输出了两个同步代码：script start，script end。

| **Task queue** | **Microtask queue** |
| -------------- | ------------------- |
| setTimeout     | Promise.then        |

接下来，将微任务取出执行，同时产生的新的微任务，在此次执行中一并处理，顺序输出：promise1，promise2

| **Task queue** | **Microtask queue** |
| -------------- | ------------------- |
| setTimeout     | Process.then        |
|                | Promise.then2       |

最后执行下一个宏任务setTimeout，输出setTimeout

| **Task queue** | **Microtask queue** |
| -------------- | ------------------- |
| setTimeout     |                     |

## EventLoop与浏览器渲染

最后来简单介绍一点event loop与浏览器渲染、帧动画以及空闲回调的内容。还是有四个问题可以先思考一下：

1、每一轮 Event Loop 都会伴随着渲染吗？

2、resize、scroll 这些事件是何时去派发的？

3、requestAnimationFrame 在哪个阶段执行,在渲染前还是后？在 microTask 的前还是后？

4、requestIdleCallback 在哪个阶段执行？如何去执行？在渲染前还是后？在 microTask 的前还是后？

关于第一个问题，规范中也介绍了event loop每一轮执行后进行渲染的标准，宏任务执行完毕，微任务队列也被清空后，进入更新渲染阶段，这里有一个“rendering opportunity”的概念，也就是说不一定每一轮 event loop都会对应一次浏览器渲染，要根据屏幕刷新率、页面性能、页面是否在后台运行来共同决定，通常来说这个渲染间隔是固定的。所以多个task很可能在一次渲染之间执行。

浏览器会尽可能的保持帧率稳定，例如页面性能无法维持 60fps（每 16.66ms 渲染一次）的话，那么浏览器就会选择 30fps 的更新速率，而不是偶尔丢帧。如果浏览器上下文不可见，那么页面会降低到 4fps 左右甚至更低。

如果满足以下条件，也会跳过渲染：

1. 浏览器判断更新渲染不会带来视觉上的改变。

2. “map of animation frame callbacks”为空，也就是帧动画回调为空，可以通过 “requestAnimationFrame”来请求帧动画。

如果浏览器判断本轮需要进行渲染，则进行下面几步：

•    对于需要渲染的文档，如果窗口的大小发生了变化，执行监听的 resize 方法。

•    对于需要渲染的文档，如果页面发生了滚动，执行 scroll 方法。

•    对于需要渲染的文档，执行帧动画回调，也就是 requestAnimationFrame 的回调。

•    对于需要IntersectionObserver 的回调。

•    对于需要渲染的文档，重新渲染绘制用户界面。

•    判断 task队列和microTask队列是否都为空，如果是的话，则进行 Idle 空闲周期的算法，判断是否要执行 requestIdleCallback 的回调函数。

从这个过程可以看出，对于resize 和 scroll来说，并不是到了这一步才去执行滚动和缩放，那岂不是要延迟很多？浏览器当然会立刻帮我们滚动视图，根据CSSOM 规范所讲，浏览器会保存一个 pending scroll event targets，等到事件循环中的 scroll这一步，去派发一个事件到对应的目标上，驱动它去执行监听的回调函数而已。resize也是同理。

在Web应用中，实现动画效果的方法比较多，Javascript 中可以通过定时器 setTimeout 来实现，css3 可以使用 transition 和 animation 来实现，html5 中的 canvas 也可以实现。除此之外，html5 还提供一个专门用于请求动画的API，那就是 requestAnimationFrame，顾名思义就是请求动画帧。requestAnimationFrame是浏览器用于定时循环操作的一个接口，类似于setTimeout，主要用途是按帧对网页进行重绘。

- 在重新渲染前调用。官方推荐的用来做一些流畅动画所应该使用的 API

- 系统来决定回调函数的执行时机，保证回调函数在屏幕每一次的刷新间隔中只被执行一次

为什么要在重新渲染前去调用？因为requestAnimationFrame是官方推荐的用来做一些流畅动画所应该使用的 API，做动画不可避免的会去更改DOM，而如果在渲染之后再去更改 DOM，那就只能等到下一轮渲染机会的时候才能去绘制出来了，这显然是不合理的。requestAnimationFrame在浏览器决定渲染之前给你最后一个机会去改变 DOM 属性，然后很快在接下来的绘制中帮你呈现出来，所以这是做流畅动画的不二选择。

相比较setTimeout的缺陷：

- setTimeout的执行时间并不是确定的，实际执行时间会比设定时间晚一些

- 刷新频率受屏幕分辨率和屏幕尺寸的影响，因此不同设备的屏幕刷新频率可能会不同，而 setTimeout只能设置一个固定的时间间隔，这个时间不一定和屏幕的刷新时间相同

requestAnimationFrame由系统来决定调用渲染的时间，相比setTimoeout更加可靠。

现在来看一个闪烁动画的小例子，假设我们现在想要快速的让屏幕上闪烁 红、蓝两种颜色，保证用户可以观察到，如果我们用 setTimeout 来写，并且带着我们长期的误解宏任务之间一定会伴随着浏览器绘制，那么你会得到一个预料之外的结果。

可以看出这个结果是非常不可控的，如果这两个 Task 之间正好遇到了浏览器认定的渲染机会，那么它会重绘，否则就不会。由于这俩宏任务的间隔周期太短了，所以很大概率是不会的。如果你依赖这个 API 来做动画，那么就很可能会造成「掉帧」。

接下来我们换成requestAnimationFrame试试？我们用一个递归函数来模拟 10 次颜色变化的动画。浏览器会非常规律的把这 10 组也就是 20 次颜色变化绘制出来。

这里是requestAnimationFrame的浏览器兼容情况，如果在开发中有场景需要这个功能，可以考虑使用这个API。

requestIdleCallback是浏览器提供给我们的空闲调度算法，关于它的简介可以看 MDN 文档，意图是让我们把一些计算量较大但是又没那么紧急的任务放到空闲时间去执行。不要去影响浏览器中优先级较高的任务，比如动画绘制、用户输入等等。

当渲染有序进行，这种有序的 “浏览器 -> 用户 -> 浏览器 -> 用户”的调度基于一个前提，就是我们要把任务切分成比较小的片，不能说浏览器把空闲时间让给你了，你去执行一个耗时“10s”的任务，那肯定也会把浏览器给阻塞住的。这就要求我们去读取requestIdleCallback提供给你的deadline里的时间，去动态的安排我们切分的小任务。浏览器信任了你，你也不能辜负它呀。

当渲染长期空闲，可能在几帧的时间内浏览器都是空闲的，并没有发生任何影响视图的操作，它也就不需要去绘制页面，这种情况下还是会有 `50ms` 的 `deadline` 呢？是因为浏览器为了提前应对一些可能会突发的用户交互操作，比如用户输入文字。如果给的时间太长了，你的任务把主线程卡住了，那么用户的交互就得不到回应了。50ms 可以确保用户在无感知的延迟下得到回应。

这里可以做一个小实验：这个动画的例子很简单，就是利用在requestAnimationFrame每帧渲染前的回调中把方块的位置向右移动 10px。最后加了一个 requestIdleCallback 的函数，回调里会 alert('rIC')，来看一下演示效果：alert 在最开始的时候就执行了，为什么会这样呢一下，想一下「空闲」的概念，我们每一帧仅仅是把 left 的值移动了一下，做了这一个简单的渲染，没有占满空闲时间，所以可能在最开始的时候，浏览器就找到机会去调用 rIC 的回调函数了。

我们简单的修改一下 step 函数，在里面加一个很重的任务，1000 次循环打印。

其实和我们预期的一样，由于浏览器的每一帧都"太忙了",导致它真的就无视我们的 rIC 函数了。

如果给 rIC 函数加一个 timeout 呢：浏览器会在大概 500ms 的时候，不管有多忙，都去强制执行 rIC 函数，这个机制可以防止我们的空闲任务被“饿死”。

最后总结一下：

事件循环不一定每轮都伴随着重渲染，但是一定会伴随着微任务执行。

决定浏览器视图是否渲染的因素很多，浏览器是非常聪明的。

requestAnimationFrame在重新渲染屏幕之前执行，非常适合用来做动画。

requestIdleCallback在渲染屏幕之后执行，并且是否有空执行要看浏览器的调度，如果你一定要它在某个时间内执行，请使用 timeout参数。

resize和scroll事件其实自带节流，它只在 Event Loop 的渲染阶段去执行事件。