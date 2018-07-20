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

## 举个例子🌰🌰🌰
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

