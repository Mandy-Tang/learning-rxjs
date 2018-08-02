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

## BehaviorSubject

## ReplaySubject

## AsyncSubject
