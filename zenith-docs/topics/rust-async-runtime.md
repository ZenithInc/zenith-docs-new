# 深入浅出 Rust 异步运行时

Rust 并没有和 Go 一样，标准库没有内置异步运行时。这是非常明智的，让这门语言可以从事更加底层的系统级开发。

同时，社区开发了 Tokio 这样的异步运行时，如今也渐渐成熟到成为标准。让 Rust 也可以擅长应用层的、IO 密集型的任务。

这篇文档将深入讲解 Rust 的异步运行时的原理，它由三部分组成，Future、Waker、Executor。通过简单的示例解释三者之间的关系以及调用逻辑。让读者深入浅出理解 Rust 中的异步运行时。

### 深入理解 Future

Rust 虽然没有实现异步运行时，但是提供了实现了一系列的 trait。其核心就是 `Future`, 定义如下:

```rust
pub trait Future {
    type Output;
    fn poll(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Self::Output>;
}
```

Future 表示一个异步计算的值，承诺在将来会返回一个值。两行代码，简洁有力:

*   `type Output`: 执行 `poll` 方法后的返回值，实现 `Future` 的结构可以自行定义这个返回值的类型。
*   `poll` 方法的签名就有一些复杂了，如下图所示:

![bc9423ec5b7718538186a76f49f5cf11.png](https://i.miji.bid/2025/04/25/bc9423ec5b7718538186a76f49f5cf11.png)

我说了那么多，你理解 Future 到底是一个什么东西了吗？反正我自己是不理解的。我们来看下面这段代码:

```rust
async fn do_something() {
    a().await;
    b().await;
}
```

编译器会将一个 `async` 块编译为一个 Future， 而 Future 本质上是一个状态机，当中包含了多个状态，每次执行 `poll` 方法会尝试推进一次状态，直到完成。就上面这个示例而言，可能会包含如下状态:

*   Start
*   WaitingA(aFuture)
*   WaitingB(bFuture)
*   Done

每次调用 `poll` 方法的时候，会尝试推进一个状态，但是如果下一个状态没有就绪的话，会继续等待, 直到所有的 `await` 都完成，状态机会进入 Done 状态，poll 会返回 `Ready`, 整个 Future 会执行完毕。

到现在为止，我们理解了 `poll` 这个方法的作用以及 `Poll` 枚举的含义，它是一个状态机(Future)对外暴露的状态，可能是 Pending 可能是 Ready。

但是，我们还不理解 `Context` 的作用，你透过源码发现其包含了 `Waker` 结构可以唤醒异步任务，但是为什么可以唤醒呢？是如何唤醒的呢？不知道，到现在为止我们所知还不足以解释它们。接下来，我们将深入异步运行时的组成，之后再来理解 Waker 的原因，会容易一些。

### Runtime 的组成

一个异步运行时由三部分组成，分别是 Future、Waker 以及 Executor。上文中，我们已经理解 Future，但是我们没有深入说 Waker 的具体实现，是因为要先理解 Executor。

Executor 包含了一个就绪队列, 保存着将要执行的 Task，一个可以调度的异步任务单元。而 Task 持有一个 Future。简化的示例代码如下:

```rust
struct Task {
    future: Pin<Box<dyn Future<Output = ()> + 'static>>,
}
struct Executor {
    ready_queue: VecDeque<Task>,
}
```

这个 Task 就负责将 Future 和唤醒机制结合起来。看如下代码:

```rust
pub struct Context<'a> {
    waker: &'a Waker,
}
impl Executor {
    fn run(&mut self) {
        while let Some(mut task) = self.ready_queue.pop_front() {
            // 这里是关键，Task 创建了 Waker，并在 Waker 中包含了 self 和 task
            let waker = task.create_waker(self);
            let mut cx = Context::from_waker(&waker);
            // 调用 future 并传入 Context
            match task.future.as_mut().poll(&mut cx) {
                Poll::Pending => {
                    // 注册事件，等待被唤醒
                    // 唤醒之后，Waker 将任务从新推入 ready_queue
                }
                Poll::Ready(_) => {/* 任务完成 */}
            }
        }
    }
}
```

通过这个示例我们可以看到：

*   **Executor 负责调度和运行任务**，具体来说就是通过从 Ready Queue 冲获取任务，然后调用 task 中的 future 的 poll 方法，推进状态的变更。

*   **Future 实现了 poll 方法**，通过 poll 方法返回 Pending 或者 Ready，推进内部状态的变更。

*   **Waker 是 Context 的核心**，负责唤醒任务，在事件发生后（比如 Epoll 等多路复用器就是事件源），之后将 future 推入 Ready Queue。而 Context 的核心作用就是传递 Waker，让 Future 能注册唤醒机制。Waker 本质上是一个事件回调。

### 总结

究其根本，异步运行时的本质，就是通过 Future、Executor 和 Waker 这三者的协作，实现了高效的非阻塞任务调度。


