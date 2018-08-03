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
