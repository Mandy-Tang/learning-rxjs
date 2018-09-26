# 使用弹珠图测试 RxJS

我们可以利用 TestScheduler 虚拟化时间，以同步地测试我们的异步 RxJS 代码。ASCII 弹珠图为我们提供了一种可视化的方法来表示可观察对象的行为，利用弹珠图，我们可以断言一个可观察对象按照我们的期望运行。

> 当前，TestScheduler 只能用于测试使用 timers 的代码，例如 delay/debouceTime/etc。如果代码使用一个 Promise，或者使用了 AsapScheduler/AnimationFrameScheduler 等，并不能通过 TestScheduler 进行可靠的测试，而应该用传统的方法测试。

```
import { TestScheduler } from 'rxjs/testing';

const scheduler = new TestScheduler((actual, expected) => {
  // asserting the two objects are equal
  // e.g. using chai.
  expect(actual).deep.equal(expected);
});

// This test will actually run *synchronously*
it('generate the stream correctly', () => {
  scheduler.run(helpers => {
    const { cold, expectObservable, expectSubscriptions } = helpers;
    const e1 =  cold('-a--b--c---|');
    const subs =     '^----------!';
    const expected = '-a-----c---|';

    expectObservable(e1.pipe(throttleTime(3, scheduler))).toBe(expected);
    expectSubscriptions(e1.subscriptions).toBe(subs);
  });
});
```

## API

## 弹珠图语法

在 TestScheduler 的上下文中，弹珠图是一个含有特殊语义的字符串，其代表了在虚拟时间轴上发生的事件。时间按帧进行。弹珠字符串上的第一个字符代表第 0 帧或者开始。在 `testScheduler.run(callback)` 中，`frameTimeFactor` 被设置为 1，代表一帧等于一个虚拟的毫秒。

每帧代表多少虚拟毫秒依赖于 `TestScheduler.frameTimeFactor` 的值。因为历史遗留原因，只有当你的代码在 `testScheduler.run(callback)` 的回调中运行时，`frameTimeFactor` 才为 1。 在这个回调之外，它的值为 10。在之后的 RxJS 的版本中，很有可能将其都设为 1。

> 重要：这个语义指南只适用于在新的 `testScheduler.run(callback)` 中使用，当手动使用 TestScheduler 时弹珠图的语义是不一样的，而且有些功能是不支持的（例如新的时间段语法）。

* `' '` 空格：水平空格会被忽略，可以用于垂直排列多个弹珠图。
* `'-'` 帧：一个虚拟时间帧
* `[0-9]+[ms|s|m]` 时间段：时间段语法让你可以表达一段虚拟时间。数字后面紧跟着时间单位，中间无空格。
* `|` 完成：一个可观察对象的成功完成。即可观察对象的 `complete()`。
* `#` 错误：一个可观察对象的错误中止。即可观察对象的 `error()`。
* `[a-z0-9]` 任意字母或数字：代表可观察对象发出的一个值，即 `next()`。
* `'()'` 同步组：当多个事件需要在一个时间帧同步时，括号用于组合这些时间。可以用这种方式组合 next，完成，或者错误。括号开始的位置代表值发生的时间。当所有的值同步发射之后，时间会前进字符串长度（包括括号）个时间帧。例如，`(abc)` 会在同一个时间帧中同时发射值 a，b，c，但是时间会向前前进五个时间帧，因为 `'(abc)'.length === 5`。
* `'^'` 订阅点：只针对热流。代表了被测试的可观察对象被热流订阅的时间点。这是热流的第 0 帧，在这之前的时间帧都是负的。负的时间看起来没有意义，但确实有存在的必要，比如涉及到 ReplaySubject 的时候。

### 时间段语法

新的时间段语法的设计灵感来源于 CSS 的 duration。这是一个数字紧跟着一个时间单位。
当它并不是弹珠图的第一个字符时，需要在它的前后都留一个空格，以消除可能带来的歧义。比如，`a 1ms b` 需要 1ms 前后的空格，因为 `a1msb` 会被解析为发射出来的一串值 `['a', '1', 'm', 's', 'b']`。

**注意**：你可能需要在你需要的时间段上减去 1ms，因为一个代表发射值的弹珠已经在其发射之后前进了一个时间帧。如下例：

```
const input = ' -a-b-c|';
const expected = '-- 9ms a 9ms b 9ms (c|)';
/*

// Depending on your personal preferences you could also
// use frame dashes to keep vertical aligment with the input
const input = ' -a-b-c|';
const expected = '------- 4ms a 9ms b 9ms (c|)';
// or
const expected = '-----------a 9ms b 9ms (c|)';

*/

const result = cold(input).pipe(
  concatMap(d => of(d).pipe(
    delay(10)
  ))
);

expectObservable(result).toBe(expected);
```

*** 例子

`'-'` 或 `'------'`：等同于 `never()`，表示一个可观察对象没有发射或者结束

`|`：等同于 `empty()`

`#`: 等同于 `throwError()`

`'--a--'`: 等待两帧之后发射值 a，然后没有结束

`'--a--b--|'`: 在第 2 帧发射值 a，在第 5 帧发射值 b，在第 8 帧结束

`'--a--b--#'`: 在第 2 帧发射值 a，在第 5 帧发射值 b，在第 8 帧发生错误

`'-a-^-b--|'`: 在一个热流上，-2 帧是发射了 a，第 2 帧发射了 b，在第 5 帧结束

`'--(abc)-|'`: 在第 2 帧发射a，b，和 c，在第 8 帧结束

`'-----(a|)`: 在第 5 帧发射值 a 和结束

`'a 9ms b 9s c|`: 在第 0 帧发射 a，在第 10 帧发射 b，在第 10,012 帧发射 c，在 10,013 帧结束

`'--a 2.5m b'`: 在第 2 帧发射 a，在第 150,003 帧发射 b 且没有结束。

## 订阅弹珠图

`expectSubscriptions` 允许你断言一个 `cold()` 或者 `hot()` 的可观察对象在正确是时间点被订阅和被取消订阅。订阅弹珠图的语法和常规的弹珠图语法相比有细微的差别。

* `'-'` 时间：经过了一个时间帧
* `[0-9]+[ms|s|m]`：时间段
* `'^'` 订阅点：订阅发生的时间点
* `'!'` 取消订阅点：订阅被取消的时间点

在一个订阅弹珠图上最多只能存在一个 `^`，也只能最多存在一个 `!`。除了这些，`-` 是唯一允许存在在订阅弹珠图上的字符。

### 例子

`'-'` or `'------'`：没有发生任何订阅

`--^--`: 在过了两帧之后订阅，且没有取消

'--^--!-': 在过了两帧之后订阅，在第 5 帧取消订阅

'500ms ^ 1s !': 在第 500 帧订阅, 在 1,501 帧取消

# 已知的问题

### 你不能直接测试消费 Promise 的代码，或者使用任何其他的 schedulers （如 AsapScheduler）

如果你的 RxJS 代码使用了 AsyncScheduler 之外的其他形式的异步 scheduler，例如 Promise，AsapScheduler 等，针对这些代码，你不能依赖弹珠图。因为这些方法不能被 TestScheduler 模拟。

解决办法是利用传统异步测试的方法隔离测试这些代码，具体的方法取决于你使用的测试框架。这里有一个伪代码的例子：

```
// Some RxJS code that also consumes a Promise, so TestScheduler won't be able
// to correctly virtualize and the test will always be really async
const myAsyncCode = () => from(Promise.resolve('something'));

it('has async code', done => {
  myAsyncCode().subscribe(d => {
    assertEqual(d, 'something');
    done();
  });
});
```

同时即便你使用了 AsyncScheduler 你也不能同步断言延迟 0，例如 `delay(0)` 相当于 `setTimeout(work, 0)`，这是异步的。

### `testScheduler.run(callback)` 的行为不一样

TestScheduler 从 v5 的版本开始就存在了，但是实际上只用于维护者们测试 RxJS 本身，而不是用于测试通常的用户应用。因此，TestScheduler 的某些默认行为和功能对用户来说并不是很好用，或者完全不能用。所以在 v6 的版本中，我们引入了 `testScheduler.run(callback)` ，用不破坏原有行为的方式提供新的预设和功能，但是仍然可以在 `testScheduler.run(callback)` 之外使用 TestScheduler。所以如果你在它之外使用，你需要知道这里有些主要的不一样的地方：

* 有一些工具方法具有更冗长的名字，例如 `testScheduler.createColdObservable()` 至于 `cold()`
* testScheduler 的实例不是被使用 AsyncScheduler 的操作符自动使用，例如 dealy，debouceTime 等，所以你需要显示地传入。
* 没有时间段语法
* 一帧默认值是10个虚拟毫秒：`TestScheduler.frameTimeFactor = 10`
* 空格和连字符都代表一帧
* 有最大帧设置 750：`maxFrames = 750`。在 750 之后都被默默忽略了
* 需要显式地 flush the scheduler
