---
layout: post
title: "Async Rust: behind the scenes"
date: 2023-01-10 00:00:00 +0530
---

# CURTAIN UP
There is a famous quote by Feynman which goes as, “If you cannot explain something in simple terms, you don't understand it”. I couldn’t agree more. And when it comes to software engineering, one can tweak it a bit and say “If you cannot implement something, you don’t understand it”.[^1] In the spirit of these quotes, here is my attempt to do both, explain and implement, how async programming actually works in Rust.[^2]

The post is born out of a simple question, “When is a Future worth polling again!?”. Don’t worry if the question doesn't make sense yet but do keep it in the back of your mind.[^3]

# MEET THE CHARACTERS
__Future__ is a basic unit which describes an async operation. It doesn't block, and returns a value immediately depending on whether a result is available or not. Let’s sketch how it would look like in Rust,

```rust
// this is simplified definition
trait Future {
    type Output;
    fn poll(self: &mut Self, waker: fn()) -> Poll<Self::Output>;
}

enum Poll<T> {
    Ready(T),
    Pending,
}
```

So it comprises two members, an output type and poll method. A result of the poll method is enum which indicates whether the result is ready or not. If it’s not, it would simply return `Poll::Pending`. Usually, in Rust, you don’t implement Future trait for a type, the compiler does that for you when you use the `async` keyword.

```rust
async fn three() -> i32 { 1 + 2 }
```

The compiler automatically wraps the result type with Future, like `Future<Output=i32>`. Following is the same definition but uses an async block rather than async function. Notice the missing `async` keyword before `fn`. 

```rust
fn three() -> impl Future<Output=i32> { async { 1 + 2 } }
```

Future’s result can be obtained by awaiting it. 

```rust
let result = three().await;
assert_eq!(result, 3);
```

But the catch is, future doesn’t execute right away (like JavaScript Promise) or by itself. This is where the __Executor__ comes into the picture. You can think of it as an orchestrator manipulating one or more futures to move them towards their result. You might have guessed by now that the executor is the one who calls Future’s `poll` method. If the result is not available, it goes to sleep (or tries to poll other queued futures, if any) until it knows that it’s a good time to poll again. But how does it know that the time is good? Who is going to “wake” the executor? Read the `poll`’s signature again.

The __Waker__ is a callback which knows how to notify the executor, and the object which implements Future knows when to call the waker.[^4]

# THE PLAY

Let’s implement these characters and perform a scene. The scene is as follows, the executor invokes timer future’s poll method, hence starts the future execution. It sees that time has not elapsed yet by inspecting its internal state and goes to sleep. Upon starting the execution, timer future will sleep for a given time duration, set internal state to indicate that it’s done sleeping and calls the waker to notify the executor about it. Because of the efficient model, the executor will poll only two times whether the given time duration is two seconds or two minutes.

#### Timer future

```rust
struct SharedState {
    completed: bool,
    waker: Option<Waker>,
}

struct Timer {
    shared_state: Arc<Mutex<SharedState>>
}

impl Future for Timer {
    type Output = ();

    fn poll(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Self::Output> {
        let mut shared_state = self.shared_state.lock().unwrap();
        if shared_state.completed {
            Poll::Ready(())
        } else {
            shared_state.waker = Some(cx.waker().clone()); // set waker
            Poll::Pending
        }
    }
}

```

The implementation is pretty straightforward. The `Timer` keeps its internal state behind mutex to avoid race conditions. The internal state is self-explanatory. The poll method gets an exclusive access to state, returns `Poll::Ready` if future is completed, otherwise stores waker in state and returns`Poll::Pending`.

But wait, what’s `Pin` and `Context`? Where is the waker callback argument? As mentioned, `waker: fn()` was a simplified representation. The `Context` bundles a bunch of information related to an asynchronous task which includes waker too. The `Pin` would require much more details to explain but just know that it’s fine to move the future around before the executor polls it. Once it’s polled, it shouldn’t be moved in memory otherwise it would hold dangling pointers. The `Pin` type restricts how the pointers are used which makes sure that it stays valid.[^5]

Who holds the knowledge that the time is elapsed and it should be notified to the executor about it? In our play, the timer itself. Let’s implement the timer constructor. It will spawn a thread and upon waking up after a given duration it sets the internal state completed and calls waker. Simple!

```rust
impl Timer {
    pub fn new(duration: Duration) -> Self {
        let shared_state = Arc::new(Mutex::new(SharedState {
            completed: false,
            waker: None,
        }));
        let thread_shared_state = shared_state.clone();
        thread::spawn(move || {
            thread::sleep(duration);
            let mut shared_state = thread_shared_state.lock().unwrap();
            shared_state.completed = true;
            if let Some(waker) = shared_state.waker.take() {
                waker.wake(); // call waker
            }
        });
        Timer { shared_state }
    }
}
```

#### Executor

Our executor has two supporting characters, task and spawner. The task is a wrapper around a future with some additional information. The spawner takes a future, creates a task out of it and submits it to the executor. In our example, the executor uses a channel to receive tasks. The other end of the channel will be used by the spawner to submit futures.

```rust
struct Task {
    future: Mutex<Option<BoxFuture<'static, ()>>>,
}

struct Spawner {
    task_sender: SyncSender<Arc<Task>>,
}

impl Spawner {
    fn spawn(&self, future: impl Future<Output = ()> + 'static + Send) {
        let future = future.boxed();
        let task = Arc::new(Task {
            future: Mutex::new(Some(future)),
        });
        self.task_sender.send(task).expect("too many tasks queued");
    }
}
```

The executor will wait for receiving a task, pulls out the future, creates context using waker and tries to poll the future. If it doesn’t complete, it puts the future back in task. Empty future in a task means the task completed with a result.

```rust
struct Executor {
    ready_queue: Receiver<Arc<Task>>,
}

impl Executor {
    fn run(&self) {
        while let Ok(task) = self.ready_queue.recv() {
            let mut future_slot = task.future.lock().unwrap();
            if let Some(mut future) = future_slot.take() {
                let waker = waker_ref(&task);
                let context = &mut Context::from_waker(waker.deref());
                if future.as_mut().poll(context).is_pending() {
                    *future_slot = Some(future);
                }
            }
        }
    }
}
```

The only puzzling thing would be the following line,
```rust
let waker = waker_ref(&task);
```
The definition of task is simple struct. What is `waker_ref`? And how does it know how to create a waker out of task? We are using the [futures](https://crates.io/crates/futures) crate which provides a trait called `ArcWake`. It has the capability to create a waker from any object of type `Arc<T>`. And the `waker_ref` method takes a reference of `Arc<ArcWake>` and returns a reference of waker `WakerRef`. 

Let’s implement `ArcWake` for the `Task` struct. It provides implementation on what to do when someone calls a waker method. The executor expects to receive tasks from the channel which are ready to be polled. Currently only the spawner has capability to send a task. Let’s update our `Task` definition to hold the sending part of the channel, and implement waker to just pass the clone of the task to the executor. The executor will handle the remaining part.

```rust
struct Task {
    future: Mutex<Option<BoxFuture<'static, ()>>>,
    task_sender: SyncSender<Arc<Task>>,
}

impl ArcWake for Task {
    fn wake_by_ref(arc_self: &Arc<Self>) {
        let cloned = arc_self.clone();
        arc_self
            .task_sender
            .send(cloned)
            .expect("too many tasks queued");
    }
}

// update the spawn method to set the `task_sender` of the Task object.
```

The dialogue between the executor and the future ultimately progresses the work: the executor polls the future and goes to sleep, then the future invokes the waker, so the executor wakes up and polls the future again.

See [the gist](https://gist.github.com/heyrutvik/42e7992f7d7d282f4124c57a226d6b68) for the complete code.

# BEYOND THE STAGE

So far so good. Best thing to do after wrapping your head around a new concept, is to read actual code that implements it in the wild. So I looked into the [async_std](https://crates.io/crates/async-std) crate. Specifically, I was interested in the network communication part. The crate depends on another crate called [async_io](https://crates.io/crates/async-io) which implements async networking types. Let’s assume following is the code we are dealing with,

```rust
task::block_on(async || {
    // let mut socket = ...
    // socket.write_all(...
    // socket.shutdown(...
    let mut resp = String::new();
    socket.read_to_string(&mut resp).await?;
    // Ok(resp)
});
```

The `block_on` method is executor. It blocks the current thread until it gets a result from the future. Here, we are trying to establish a socket connection, writing some data to it and waiting for a response from the remote server. Let’s assume data was not available when the executor first polled the `read_to_string` future. It then goes to sleep. How does the executor know when to poll again?

All modern operating systems provide some mechanism to watch a list of file descriptors and notify the process which file descriptors are ready to perform I/O operations. Linux provides epoll, macOS provides kqueue and Windows provides wepoll. The [polling](https://crates.io/crates/polling) crate provides an abstraction over OS-specific implementations.

The idea is the same. An object of [Async&lt;T&gt;](https://github.com/smol-rs/async-io/blob/master/src/lib.rs#L594) holds information about socket’s [file descriptor](https://github.com/smol-rs/async-io/blob/master/src/reactor.rs#L346) and [waker](https://github.com/smol-rs/async-io/blob/master/src/reactor.rs#L369). A separate thread registers those file descriptors to [watch for either read or write operation](https://github.com/smol-rs/async-io/blob/master/src/reactor.rs#L270) to perform. Once it receives a notification from the kernel, it [invokes waker](https://github.com/smol-rs/async-io/blob/master/src/reactor.rs#L326) associated with a file descriptor which [wakes the executor](https://github.com/smol-rs/async-io/blob/master/src/driver.rs#L133) thread and [tries to poll](https://github.com/smol-rs/async-io/blob/master/src/driver.rs#L146) the future again.

# BIBLIOGRAPHY
1. [Programming Rust, 2nd Edition](https://www.oreilly.com/library/view/programming-rust-2nd/9781492052586/) by Jim Blandy et al.
2. [Asynchronous Programming in Rust](https://rust-lang.github.io/async-book/)
3. Crate async-io
4. Crate polling

# NOTES
[^1]: I know that's a bit of an exaggeration. I can explain to someone how Docker works but implementing it would require a lot more “understanding” and time.
[^2]: The blog post explains the mechanics of async code in Rust and does not cover why/when one should use async programming.
[^3]: The blog post expects an intermediate level of Rust knowledge.
[^4]: The latter part of the statement is not _entirely_ true. It is true in our example but different executor architecture can employ different strategies.
[^5]: I think `Pin` would require a separate blog post. 
