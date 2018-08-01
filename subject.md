# Subject

`Subject` 是一类特殊的 `Observable`，普通的 `Observable` 并不具有多路推送的能力（每一个 `Observable` 都有自己独立的执行环境），而 `Subject` 可以共享一个执行环境。

`Subject` 继承自 `Observable`, 与 `Observable` 不同的的地方在于，其有一个 `observers: Observer<T>[]` 的属性，维护着一个 subscribe 了它的 `Observer` 的数组。

`Subject` 具有方法 `next(value?: T)`， `error(err: any)`，`complete()` 等，所以 `Subject` 也可以作为 `Observer` 使用。在调用 `next(value)` 方法之后，`Subject` 会向所有已经在其上注册的 `Observer` （即其实例上维护的observers 数组）多路推送 `value`。

`Subject` 作为 `Observable`:
```
const subject$ = new Subject();
subject$.subscribe(data => {
  console.log(`observerA: ${data}`);
});
subject$.subscribe(data => {
  console.log(`observerB: ${data}`);
});
subject$.next(1);
subject$.next(2);
```