# RxJS概述

## 简介
> RxJS is a library for composing asynchronous and event-based programs by using observable sequences.
RxJS是一个通过**可观察对象序列**，用于**流水线式**处理**异步操作和事件**的前端开发工具库。它支持我们像处理数据集合一样去处理异步事件。
RxJS的核心包括：
- 一个核心类型：可观察对象（Observable）
- 三个次要类型：观察者（Observer）、调度器（Schedulers）、主题（Subject）
- 类似于数组处理的操作符（`map`，`filter`，`reduce`，`every` 等等）
我们可以把 RxJS 当做一个专门用于处理异步事件的 Lodash 库。

## 基本概念和术语

- **可观察对象（Observable）**：一系列未来的值或者事件的集合
- **观察者（Observer）**：一个回调集合，包含了如何处理可观察对象传递来的值的逻辑
- **订阅（Subscription）**：代表了一个可观察对象的执行，主要用于关闭一个执行
- **操作符（Operators）**：处理集合的纯函数
- **主题（Subject）**：等同于一个事件发射器，支持多播，即可以向多个观察者发射值
- **调度器（Schedulers）**：一个管理并发的集中式分发器，使我们可以处理 `setTimeout`, `requestAnimationFrame` 等

## 举个官网上的例子🌰🌰🌰
在前端开发的过程中，我们通常使用下面的方式去注册一个事件监听
```
const button = document.querySelector('button')
button.addEventListener('click', () => console.log('Clicked!'))
```
使用 RxJS，我们可以创建一个可观察对象并订阅它
```
const { fromEvent } = rxjs;

const button = document.querySelector('button')
fromEvent(button, 'click')
  .subscribe(() => console.log('Clicked!'))
```
在这个例子中，`fromEvent(button, 'click')` 创建了一个可观察对象，在未来为发射一系列点击事件，`subscribe` 表示订阅这个可观察对象，`() => console.log('Clicked!')` 这个回调函数就是一个观察者，包含了对每一个事件的处理逻辑，这里的处理逻辑只是一个简单的 `log`，在我们的实际应用中，这里会按照需求包含各种各样的业务逻辑代码，`fromEvent(button, 'click')
  .subscribe(() => console.log('Clicked!'))` 的返回值就是一个订阅，在特定的时候我们可以取消这个订阅，当然在这段代码中并没有这个操作。

## RxJS提倡纯函数
RxJS提供了大量的纯函数（pure function），纯函数的好处有很多，其中一点就是对外部状态没有依赖，减少了对外部环境的污染，从而可以减少发生错误的可能性。
再举个官网上的例子🌰🌰🌰，基于之前的例子，我们要增加点击计数的功能。传统的事件监听的写法如下：
```
var count = 0
var button = document.querySelector('button')
button.addEventListener('click', () => console.log(`Clicked ${++count} times`))
```
我们需要在回调函数的外部去定义一个变量 `count` 用于统计点击次数。而这个变量其实在外部并没有作用，只用于这个回调函数。这样在这个外部作用域中，不仅多了一个 `count` 变量，污染了外部作用域；而且，一段时间之后，如果我们需要把这个时间监听挪到另一个文件中，但是忘记同时把 `var count = 0` 一起挪走，那么我们的程序就会报错。

如果用 RxJS，写法如下：
```
const { fromEvent } = rxjs
const { scan } = rxjs.operators

const button = document.querySelector('button')
fromEvent(button, 'click').pipe(
  scan(count => count + 1, 0)
  ).subscribe(count => console.log(`Clicked ${count} times`))
```
在这里，`scan` 的含义和操作数组的 `reduce` 类似，前一次回调函数的返回值会作为下一次回调函数的输入。然而和传统事件监听的写法相比，`count`在这个数据流的内部产生并暴露给观察者，不依赖于任何外部环境的值。

## RxJS提供了丰富的控制流的操作符
再再再举个官网上的例子🌰🌰🌰。在之前的例子的基础上，如果我们要给点击事件加上额外的需求：只有当两次点击事件的时间间隔不小于1000毫秒时，才技术。没有RxJS的情况下我们会这样子写代码：
```
var count = 0;
var rate = 1000;
var lastClick = Date.now() - rate;
var button = document.querySelector('button');
button.addEventListener('click', () => {
  if (Date.now() - lastClick >= rate) {
    console.log(`Clicked ${++count} times`);
    lastClick = Date.now();
  }
});
```
增加 `lastClick` 变量用于记录最近一次点击事件的时间，在时间监听回调函数里面判断与当前的时间间隔是否满足条件，若满足则计数，且更新 `lastClick`。
那么如果用 RxJS ，这段逻辑可以这样写：
```
const { fromEvent } = rxjs;
const { throttleTime, scan } = rxjs.operators;

const button = document.querySelector('button');
fromEvent(button, 'click').pipe(
  throttleTime(1000),
  scan(count => count + 1, 0)
)
.subscribe(count => console.log(`Clicked ${count} times`));
```
`throttleTime(1000)` 这里直接使用了 RxJS 提供的操作符 `throttleTime`，表示忽略数据流中时间间隔小于1000毫秒的值。使用简单且逻辑一目了然。RxJS为开发者提供了丰富的流控制操作符，具体请阅读 API 文档。

## RxJS提供了丰富的数据处理操作符
还是之前的例子，假设需求要求累积每次点击事件的X坐标，而不是点击的次数。事件监听回调函数接收当次事件为参数，在回调函数内部获取事件的X坐标，累积到 `count`。
```
let count = 0;
const rate = 1000;
let lastClick = Date.now() - rate;
const button = document.querySelector('button');
button.addEventListener('click', (event) => {
  if (Date.now() - lastClick >= rate) {
    count += event.clientX;
    console.log(count)
    lastClick = Date.now();
  }
});
```
使用 RxJS 我们可以这样写：
```
const { fromEvent } = rxjs;
const { throttleTime, map, scan } = rxjs.operators;

const button = document.querySelector('button');
fromEvent(button, 'click').pipe(
  throttleTime(1000),
  map(event => event.clientX),
  scan((count, clientX) => count + clientX, 0)
)
.subscribe(count => console.log(count));
```
`map` 对数据流中的每个值进行映射，返回坐标，并传递给下一个操作符。`scan` 做一个类似于 `reduce` 的操作，接收上一次的计算结果以及当前的值，经过计算之后，再发射给观察者。观察者接收到的是已经处理好的数据。
RxJS 提供了大量类似的 API 以方便开发者做数据处理。我们可以看到这里的数据处理逻辑和函数式编程工具库 `Ramdajs` 的写法和 API 高度相似，`pipe` 和 `map` 我们都可以在 `Ramdajs` 中找到相同的 API，`scan` 对应的就是 `reduce`，这样我们是不是更能够理解 RxJS 可以像处理数据集合一样去处理异步事件？是不是更能体会到 RxJS 的纯函数特点？