# execute&submit

1. execute 就是主线程让线程池去做事 如果做事发生异常了 直接抛出异常 整个程序都知道了
2. submit 主线程让线程池去做事 但是会有返回值。主线程交代后，线程池会先给主线程一个Future对象，相当于小票，然后主线程就去忙别的了。如果线程池发生异常，会把异常记录在Future对象，只有主线程主动调用future.get()，才会知道发生异常。

## 实际核心区别

 1. 接受的参数不同 (`Runnable` vs `Callable`)

- `execute` 只能吃 `Runnable`。它的 `run()` 方法没有返回值，也不能抛出受检异常（Checked Exception）。
- `submit` 既能吃 `Runnable`，也能吃 `Callable`。`Callable` 的 `call()` 方法是有返回值的，而且允许抛出异常。

#### 2. 返回结果不同 (`void` vs `Future`)

- `execute` 的返回值是 `void`，任务扔进去就像泥牛入海。
- `submit` 会返回一个 `Future` 接口的实现类。你可以通过 `Future.get()` 阻塞等待任务完成并拿到结果，也可以用 `Future.cancel()` 取消任务。

#### 3. ⚠️ 异常处理机制（大厂最高频挖坑点）

- **`execute`**：任务内部抛出未捕获的运行时异常（RuntimeException）时，**当前执行任务的线程会直接崩溃（死掉）**。随后线程池会捕获这个异常并打印出来，然后再默默新建一个线程来补充空缺。
- **`submit`**：底层会把任务包装成一个 `FutureTask`。如果任务抛出异常，线程池底层的 catch 块会把异常**吞掉**，并保存到 Future 对象中。**执行任务的线程不会死**。只有当你调用 `Future.get()` 时，它才会把当初的异常包装成 `ExecutionException` 重新抛给你。

## 提高

如果在使用 `submit` 时，不需要处理返回值，也**一定不要忘记处理异常**。最佳实践是在 `Callable` 或 `Runnable` 的业务代码最外层，加上 `try-catch` 兜底，防止因为没有调用 `Future.get()` 而导致线上故障被“静默吞噬”（也就是所谓的“异常丢失”）。





