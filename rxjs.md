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

## RxJS提供了很多功能的操作符以满足开发者对数据流的控制
