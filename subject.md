# Subject

`Subject` 是一类具有多播功能的特殊 `Observable`，普通的 `Observable` 并不具有多路推送的能力（每一个 `Observable` 都有自己独立的执行环境），而 `Subject` 可以共享一个执行环境。

`Subject` 继承自 `Observable`, 与 `Observable` 不同的的地方在于，其有一个 `observers: Observer<T>[]` 的属性，维护着一个 subscribe 了它的 `Observer` 的数组。（类似于 EventEmitters：维护了一个 listeners 的列表）

**Subject 是 Observable**，我们可以用一个 Observer 去 `subscribe` 一个它，并开始接收它发出的值。从 Observer 的角度来看，并不能分辨其 `subscribe` 的是 `Subject` 还是 `Observable`。

**Subject 也是 Observer**，`Subject` 具有方法 `next(value?: T)`， `error(err: any)`，`complete()` 等。在调用 `next(value)` 方法之后，`Subject` 会向所有已经在其上注册的 `Observer` （即其实例上维护的observers 数组）多路推送 value。

`Subject` 作为 `Observable`:
```
var subject = new Rx.Subject();

subject.subscribe({
  next: (v) => console.log('observerA: ' + v)
});
subject.subscribe({
  next: (v) => console.log('observerB: ' + v)
});

subject.next(1);
subject.next(2);
```
输出结果应为：
```
observerA: 1
observerB: 1
observerA: 2
observerB: 2
```

`Subject` 也可以作为 `Observer`，上例中的两次 next 可以用 subscribe 一个 Observable代替：
```
var subject = new Rx.Subject();

subject.subscribe({
  next: (v) => console.log('observerA: ' + v)
});
subject.subscribe({
  next: (v) => console.log('observerB: ' + v)
});

var observable = Rx.Observable.from([1, 2]);

observable.subscribe(subject); 
```
输出结果同上。
通过这个例子，我们可以发现，利用 Subject 可以把单播的 Observable 转变成了多播。
Subeject 有三个衍生类，`BehaviorSubject`，`ReplaySubject`，`AsyncSubject`。

## 多播的 Observables

多播的 Observable 利用 Subject 向多个 Observers 推送消息，而单播 Observable 只能向一个 Observer 推送消息。
多播的 Observable 利用一个 Subject 让多个 Observers 看到相同的执行环境。

其实，这就是 `multicast` 操作符是如何工作的：Observer 订阅了一个 Subject，而这个 Subject 订阅了一个原始的 Observable。下面这个例子和之前 `observable.subscribe(subject)` 的例子的功能类似。

```
var source = Rx.Observable.from([1, 2, 3]);
var subject = new Rx.Subject();
var multicasted = source.pipe(multicast(subject));

// These are, under the hood, `subject.subscribe({...})`:
multicasted.subscribe({
  next: (v) => console.log('observerA: ' + v)
});
multicasted.subscribe({
  next: (v) => console.log('observerB: ' + v)
});

// This is, under the hood, `source.subscribe(subject)`:
multicasted.connect();
```

`multicast` 返回了一个 `ConnectableObservable`，它是一个 Observable，但当被 subsribe 的时候，功能和 Subject类似。
`ConnectableObservable` 继承自 Observable，但其有一个 `connect()` 方法，用于决定共享的 Observable 的执行环境什么时候开始。`connect()` 在底层执行的就是 `source.subscribe(subject)` 的方法，其返回的是一个 Subscription，可以通过 unsubscribe 它以停止共享 Observable 的执行环境。

### 引用计数 Reference counting

手动调用 `connect` 并处理 Subscription 非常不方便。通常情况下，我们希望在第一个 Observer 到达时自动去 connect，在最后一个 Observer ubsubscribe 的时候取消共享执行环境。
考虑下面这个例子：
1. 第一个 Observer 订阅了多路推送的 Observable
2. 多路 Observable 被连接
3. 向第一个 Observer 推送值 0
4. 第二个 Observer 订阅了多路推送的 Observable
5. 向第一个 Observer 推送值 1
6. 向第二个 Observer 推送值 1
7. 第一个 Observer 取消了对多路推送的 Observable 的订阅
8. 向第二个 Observer 推送值 2
9. 第二个 Observer 取消了对多路推送的 Observable 的订阅
10. 取消对多路推送的 Observable 的连接

通过显示调用 `connect()`，代码如下：

```
var source = Rx.Observable.interval(500);
var subject = new Rx.Subject();
var multicasted = source.pipe(multicast(subject));
var subscription1, subscription2, subscriptionConnect;

subscription1 = multicasted.subscribe({
  next: (v) => console.log('observerA: ' + v)
});
// We should call `connect()` here, because the first
// subscriber to `multicasted` is interested in consuming values
subscriptionConnect = multicasted.connect();

setTimeout(() => {
  subscription2 = multicasted.subscribe({
    next: (v) => console.log('observerB: ' + v)
  });
}, 600);

setTimeout(() => {
  subscription1.unsubscribe();
}, 1200);

// We should unsubscribe the shared Observable execution here,
// because `multicasted` would have no more subscribers after this
setTimeout(() => {
  subscription2.unsubscribe();
  subscriptionConnect.unsubscribe(); // for the shared Observable execution
}, 2000);
```

如果我们不想显示调用 `connect()`，我们可以使用 ConnectableObservable 的 `refCount()` 方法。该方法会返回一个 Observable，其会记录自己有多少个 subscribers. 当它的 subscribers 从 0 增加到 1 时，它会自动地调用 `connect()` 方法，开始共享执行环境。只有当 subscribers 的数量从 1 减少到 0 时，它会被完全地 ubsubscribe，停止之后的执行。
`refCount()` 让多播的 Observable 在有第一个 subscribe 时开始执行，在最后一个 subscribe 取消订阅时停止执行。
下面是使用 `refCount()` 的例子：
```
var source = Rx.Observable.interval(500);
var subject = new Rx.Subject();
var refCounted = source.pipe(multicast(subject), refCount());
var subscription1, subscription2;

// This calls `connect()`, because
// it is the first subscriber to `refCounted`
console.log('observerA subscribed');
subscription1 = refCounted.subscribe({
  next: (v) => console.log('observerA: ' + v)
});

setTimeout(() => {
  console.log('observerB subscribed');
  subscription2 = refCounted.subscribe({
    next: (v) => console.log('observerB: ' + v)
  });
}, 600);

setTimeout(() => {
  console.log('observerA unsubscribed');
  subscription1.unsubscribe();
}, 1200);

// This is when the shared Observable execution will stop, because
// `refCounted` would have no more subscribers after this
setTimeout(() => {
  console.log('observerB unsubscribed');
  subscription2.unsubscribe();
}, 2000);
```
输出为：
```
observerA subscribed
observerA: 0
observerB subscribed
observerA: 1
observerB: 1
observerA unsubscribed
observerB: 2
observerB unsubscribed
```
需要注意的是，`connect()` 方法只存在在 ConnectableObservable 上，且其执行之后返回的是一个 `Observable` 而不是一个 `ConnectableObservable`。


## BehaviorSubject

`BehaviorSubject` 是 Subject 的衍生类，其保留了一个当前值（即最后一次发生的值），每当有一个新的 Observer subscribe 它的时候，这个 Observer 会立即接受到这个当前值。
查看源码，BehaviorSubject 在内部保留了一个 `_value`，每当执行 `next` 时，会把当前发射的值保存在 `_value` 上，每当有一个新的 Observer subscribe 时，会调用一遍这个新的 Observer 的 `next(_value)`，所以每当有一个新的 Observer，就会立即接收到这个最近一次的值。
在下面的例子中，我们用 0 初始化了一个 BehaviorSubject，当第一个 Observer subscribe 它的时候会立即收到值 0，之后这个 subject 发射了值 1 和 2，所以第二个 Observer subsribe 它时会立马收到值 2：
```
var subject = new Rx.BehaviorSubject(0); // 0 is the initial value

subject.subscribe({
  next: (v) => console.log('observerA: ' + v)
});

subject.next(1);
subject.next(2);

subject.subscribe({
  next: (v) => console.log('observerB: ' + v)
});

subject.next(3);
```
输出应该为：
```
observerA: 0
observerA: 1
observerA: 2
observerB: 2
observerA: 3
observerB: 3
```

## ReplaySubject

`ReplaySubject` 和 `BehaviorSubject` 类似，也可以把旧的值发送给新的 subscriber，不同的点在于，BehaviorSubject只能保留最后一个值，而 ReplaySubject 可以保留最近的一部分值。

我们可以设置保留多少个值：
```
var subject = new Rx.ReplaySubject(3); // buffer 3 values for new subscribers

subject.subscribe({
  next: (v) => console.log('observerA: ' + v)
});

subject.next(1);
subject.next(2);
subject.next(3);
subject.next(4);

subject.subscribe({
  next: (v) => console.log('observerB: ' + v)
});

subject.next(5);
```
输出为：
```
observerA: 1
observerA: 2
observerA: 3
observerA: 4
observerB: 2
observerB: 3
observerB: 4
observerA: 5
observerB: 5
```
在这里设置了保留 3 个值，所以 observerB subscribe 之后可以立马收到`2,3,4`，而不能收到 1。

我们也可以设置一个窗口时间，表明保留在多久时间范围内以内的值：
```
var subject = new Rx.ReplaySubject(100, 500 /* windowTime */);

subject.subscribe({
  next: (v) => console.log('observerA: ' + v)
});

var i = 1;
setInterval(() => subject.next(i++), 200);

setTimeout(() => {
  subject.subscribe({
    next: (v) => console.log('observerB: ' + v)
  });
}, 1000);
```
输出为：
```
observerA: 1
observerA: 2
observerA: 3
observerA: 4
observerA: 5
observerB: 3
observerB: 4
observerB: 5
observerA: 6
observerB: 6
...

```
这里我们虽然设置了可以保留 100 个值，但是另外设置了 500ms 的窗口时间，在 ObserverB subscibe 的时候，ReplaySubject已经发送了值 `1,2,3,4,5`，但是只有 `4,5` 是在 500ms 之内发送的，所以只能收到 `4,5`。

## AsyncSubject

`AsyncSubject` 只会在执行结束后把最后一个执行结果发送给它的 observers。
```
var subject = new Rx.AsyncSubject();

subject.subscribe({
  next: (v) => console.log('observerA: ' + v)
});

subject.next(1);
subject.next(2);
subject.next(3);
subject.next(4);

subject.subscribe({
  next: (v) => console.log('observerB: ' + v)
});

subject.next(5);
subject.complete();
```
输出为：
```
observerA: 5
observerB: 5
```
AsyncSubject 和 `last()` 操作符类似，等待完成通知后推送执行过程的最后一个值。