title: rust 笔记 - 构建多线程 web server
author: cwen
date: 2017-10-22
update: 2017-10-22
tags:
    - 笔记
    - rust
    - web

---
入坑 rust， 学习如何来用 rust 构建多线程 web server...<!--more-->

## 先看结构

```rust
## main.rs
extern crate hello;
use hello::ThreadPool;
use std::io::prelude::*;
use std::net::TcpListener;
use std::net::TcpStream;
use std::fs::File;
use std::thread;
use std::time::Duration;

fn main() {
    let listener = TcpListener::bind("127.0.0.1:8080").unwrap();

    let pool = ThreadPool::new(4);

    let mut counter = 0;

    for stream in listener.incoming() {
        if counter == 2 {
            println!("Shutting down.");
            break;
        }
        counter += 1;

        let stream = stream.unwrap();

        pool.execute(|| {
            handle_connection(stream);
        });
    }
}

fn handle_connection(mut stream: TcpStream) {
    let mut buffer = [0;512];

    stream.read(&mut buffer).unwrap();

    let get = b"GET / HTTP/1.1\r\n";
    let sleep = b"GET /sleep HTTP/1.1\r\n";

    let (status_line, filename) = if buffer.starts_with(get) {
        ("HTTP/1.1 200 OK\r\n\r\n", "html/hello.html")
    } else if buffer.starts_with(sleep) {
        thread::sleep(Duration::from_secs(5));
        ("HTTP/1.1 200 OK\r\n\r\n", "html/hello.html")
    } else {
        ("HTTP/1.1 404 NOT FOUND\r\n\r\n", "html/404.html")
    };

    let mut file = File::open(filename).unwrap();
    let mut contents = String::new();

    file.read_to_string(&mut contents).unwrap();

    let response = format!("{}{}", status_line, contents);

    stream.write(response.as_bytes()).unwrap();
    stream.flush().unwrap();
}

```

从代码可以看出，使用 rust 构建一个 web server 并不难。 从头一点点学习
其他语言一样，要监听 TCP 端口，rust 使用 `TcpListener` 中 `bind` 函数绑定监听地址，`bind` 函数返回 `Result<T, E>`, 绑定可能会失败，例如，如果不是管理员尝试连接 80 端口。另一个绑定会失败的情况是两个程序监听相同的端口，这可能发生于运行两个本程序的实例时。`unwrap` 取出 `T`， 如果出现错误直接 `panic`。

```
let pool = ThreadPool::new(4);
```

从字母意思可以看出来， 这是创建一个线程池（自己实现的，后面会详细介绍如何实现这个线程池）。

```
for stream in listener.incoming() {
   let stream = stream.unwrap();
}
```
`TcpListener` 的 `incoming` 方法返回一个迭代器，它提供了一系列的流（更准确的说是 `TcpStream` 类型的流）。流（`stream`）代表一个客户端和服务端之间打开的连接。为此，`TcpStream` 允许我们读取它来查看客户端发送了什么，并可以编写响应。所以这个 for 循环会依次处理每个连接并产生一系列的流供我们处理。

```
fn handle_connection(mut stream: TcpStream) {
    let mut buffer = [0;512];
    stream.read(&mut buffer).unwrap();
    let get = b"GET / HTTP/1.1\r\n";
    let (status_line, filename) = if buffer.starts_with(get) {
        ("HTTP/1.1 200 OK\r\n\r\n", "html/hello.html")
    } else {
        ("HTTP/1.1 404 NOT FOUND\r\n\r\n", "html/404.html")
    };
    let mut file = File::open(filename).unwrap();
    let mut contents = String::new();
    file.read_to_string(&mut contents).unwrap();
    let response = format!("{}{}", status_line, contents);
    stream.write(response.as_bytes()).unwrap();
    stream.flush().unwrap();
}
```

`handle_connection` 负责处理请求和响应，在 `handle_connection` 中，通过 `mut` 关键字将 `stream` 参数变为可变。我们将从流中读取数据，所以它需要是可修改的。

接下来，需要实际读取流。这里分两步进行：首先，在栈上声明一个 `buffer` 来存放读取到的数据。这里创建了一个 512 字节的缓冲区，它足以存放基本请求的数据。这对于本章的目的来说是足够的。如果希望处理任意大小的请求，管理所需的缓冲区将更复杂，不过现在一切从简。接着将缓冲区传递给 `stream.read` ，它会从 `TcpStream `中读取字节并放入缓冲区中。
接下来就是进行路由判断，当然这里是最简单判断，如果请求是 "/" 将返回 hello.html 页面否则返回 "404.html"。

## 实现线程池

先看肿么用

```
let pool = ThreadPool::new(4);

    let mut counter = 0;

    for stream in listener.incoming() {

        let stream = stream.unwrap();
        pool.execute(|| {
            handle_connection(stream);
        });
    }
```

 首先使用  `new` 创建一个大小为 4 的连接池， 并且调用  `execute` 方法，`execute` 方法的参数可以看出来传入的是一个闭包，并且这个线程只会执行闭包一次。所以判断 execute 参数具有 `FnOnce trait bound`, 同时可以从 `thread::spawn` 函数的实现推断出。

```
 pub fn spawn<F, T>(f: F) -> JoinHandle<T>
    where
        F: FnOnce() -> T + Send + 'static,
        T: Send + 'static
```

 所有我没制动，我的线程池 lib 需有有 `ThreadPool` 这个 struct 并且，需要绑定  `new` `execute` 函数。

 // lib.rs 基本
```
pub struct ThreadPool;

impl ThreadPool {
    pub fn new(size: u32) -> ThreadPool {
        ThreadPool
    }
    pub fn execute<F>(&self, f: F)
        where
            F: FnOnce() + Send + 'static
    {

    }
}
```

F 是这里我们关心的参数；T 与返回值有关所以我们并不关心。考虑到 `spawn` 使用 `FnOnce` 作为 F 的 `trait bound`，这可能也是我们需要的，因为最终会将传递给 `execute` 的参数传给 `spawn`。因为处理请求的线程只会执行闭包一次，这也进一步确认了 `FnOnce` 是我们需要的 `trait`。

F 还有 `trait bound Send `和生命周期绑定 `'static`，这对我们的情况也是有意义的：需要 Send 来将闭包从一个线程转移到另一个线程，而 `'static` 是因为并不知道线程会执行多久。
`FnOnce trait` 仍然需要之后的 ()，因为这里的 `FnOnce` 代表一个没有参数也没有返回值的闭包。正如函数的定义，返回值类型可以从签名中省略，不过即便没有参数也需要括号。

显然这不是一个完整连接池，没法提供服务。 我们接续补全。 结下来我们要做的是要储存它们，显然就是储存事先创建的线程。

我们定义一个 ` Worker struct` , `ThreadPool` 里面定义一个存放 size 个元素的 vector

```
pub struct ThreadPool {
    workers: Vec<Worker>,
}

struct Worker {
    id: usize,
    thread: thread::JoinHandle<()>,
}
```

接下来给 `Worker` 添加一个 new 函数

```
impl Worker {
    fn new(id: usize) -> Worker {
        let thread = thread::spawn(|| {});

        Worker {
            id,
            thread,
        }
    }
}
```

在接下来就是考虑如何通信的问题，如何让 ThreodPool 接受到请求然后让worker 去执行，首先想到的就是使用通道了，搞个生产者，消费者了。
接下来我们要做什么
    1. ThreadPool 会创建一个通道并充当发送端。
    2. 每个 Worker 将会充当通道的接收端。
    3. 新建一个 Job 结构体来存放用于向通道中发送的闭包。
    4. ThreadPool 的 execute 方法会在发送端发出期望执行的任务。
    5. 在线程中，Worker 会遍历通道的接收端并执行任何接收到的任务。


```
// ...snip...
use std::sync::mpsc;

pub struct ThreadPool {
    workers: Vec<Worker>,
    sender: mpsc::Sender<Job>,
}

struct Job;

impl ThreadPool {
    // ...snip...
    pub fn new(size: usize) -> ThreadPool {
        assert!(size > 0);

        let (sender, receiver) = mpsc::channel();

        let mut workers = Vec::with_capacity(size);

        for id in 0..size {
            workers.push(Worker::new(id));
        }

        ThreadPool {
            workers,
            sender,
        }
    }
    // ...snip...
}

impl Worker {
    fn new(id: usize, receiver: mpsc::Receiver<Job>) -> Worker {
        let thread = thread::spawn(|| {
            receiver;
        });

        Worker {
            id,
            thread,
        }
    }
}
```

`ThreadPool` 来储存一个发送 Job 实例的通道发送端

在 `ThreadPool::new` 中，新建了一个通道，并接着让线程池在接收端等待。这段代码能够编译，不过仍有警告。

在线程池创建每个 worker 时将通道的接收端传递给他们。须知我们希望在 `worker` 所分配的线程中使用通道的接收端，所以将在闭包中引用 `receiver` 参数。

显然上边的消费者是有问题。 Rust 所提供的通道实现是多生产者，单消费者的，所以不能简单的克隆通道的消费端来解决问题。即便可以我们也不希望克隆消费端；在所有的`worker` 中共享单一 `receiver` 才是我们希望的在线程间分发任务的机制。

另外，从通道队列中取出任务涉及到修改 `receiver`，所以这些线程需要一个能安全的共享和修改 `receiver` 的方式。如果修改不是线程安全的，则可能遇到竞争状态，例如两个线程因同时在队列中取出相同的任务并执行了相同的工作。

 我们可以使用线程安全智能指针，为了在多个线程间共享所有权并允许线程修改其值，需要使用 `Arc<Mutex<T>>`。`Arc` 使得多个 `worker` 拥有接收端，而 `Mutex` 则确保一次只有一个 `worker` 能从接收端得到任务。

 so 我的代码就改成这样了


```
 use std::sync::Arc;
use std::sync::Mutex;

// ...snip...

impl ThreadPool {
    // ...snip...
    pub fn new(size: usize) -> ThreadPool {
        assert!(size > 0);

        let (sender, receiver) = mpsc::channel();

        let receiver = Arc::new(Mutex::new(receiver));

        let mut workers = Vec::with_capacity(size);

        for id in 0..size {
            workers.push(Worker::new(id, receiver.clone()));
        }

        ThreadPool {
            workers,
            sender,
        }
    }
    // ...snip...
}
impl Worker {
    fn new(id: usize, receiver: Arc<Mutex<mpsc::Receiver<Job>>>) -> Worker {
        // ...snip...
    }
}

```

让我们实现 `ThreadPool` 上的 `execute` 方法。同时也要修改 `Job` 结构体：它将不再是结构体，`Job` 将是一个有着 `execute` 接收到的闭包类型的 `trait` 对象的类型别名。在 worker 中，传递给 `thread::spawn` 的闭包仍然还只是引用了通道的接收端。但是我们需要闭包一直循环，向通道的接收端请求任务，并在得到任务时执行他们。

```

type Job = Box<FnOnce() + Send + 'static>;

impl ThreadPool {
    // ...snip...

    pub fn execute<F>(&self, f: F)
        where
            F: FnOnce() + Send + 'static
    {
        let job = Box::new(f);

        self.sender.send(job).unwrap();
    }
}


impl Worker {
    fn new(id: usize, receiver: Arc<Mutex<mpsc::Receiver<Job>>>) -> Worker {
        let thread = thread::spawn(move || {
            loop {
                let job = receiver.lock().unwrap().recv().unwrap();

                println!("Worker {} got a job; executing.", id);

                (*job)();
            }
        });

        Worker {
            id,
            thread,
        }
    }
}
```

如果现在编译我们的代码，还是会出现错误

```
error[E0161]: cannot move a value of type std::ops::FnOnce() +
std::marker::Send: the size of std::ops::FnOnce() + std::marker::Send cannot be
statically determined
  --> src/lib.rs:63:17
   |
63 |                 (*job)();
   |                 ^^^^^^
```

为了调用储存在 `Box<T> `（这正是 Job 别名的类型）中的 `FnOnce` 闭包，该闭包需要能将自己移动出 `Box<T>`，因为当调用这个闭包时，它获取 `self` 的所有权。通常来说，将值移动出 `Box<T>` 是不被允许的，因为 Rust 不知道 `Box<T>` 中的值将会有多大。

我们可以使用 `self: Box<Self> `语法的方法，获取了储存在 `Box<T>` 中的 `Self` 值的所有权。这正是我们希望做的，然而不幸的是 Rust 调用闭包的那部分实现并没有使用 `self: Box<Self>`。所以这里 Rust 也不知道它可以使用 `self: Box<Self>` 来获取闭包的所有权并将闭包移动出 `Box<T>`。

不过目前让我们绕过这个问题。所幸有一个技巧可以显式的告诉 Rust 我们处于可以获取使用 `self: Box<Self>` 的 `Box<T>` 中值的所有权的状态，而一旦获取了闭包的所有权就可以调用它了。这涉及到定义一个新 `trait`，它带有一个在签名中使用 `self: Box<Self>` 的方法 `call_box`，为任何实现了 `FnOnce() `的类型定义这个 `trait`，修改类型别名来使用这个新 `trait`，并修改 `Worker` 使用 `call_box` 方法

```
trait FnBox {
    fn call_box(self: Box<Self>);
}

impl<F: FnOnce()> FnBox for F {
    fn call_box(self: Box<F>) {
        (*self)()
    }
}

type Job = Box<FnBox + Send + 'static>;

// ...snip...

impl Worker {
    fn new(id: usize, receiver: Arc<Mutex<mpsc::Receiver<Job>>>) -> Worker {
        let thread = thread::spawn(move || {
            loop {
                let job = receiver.lock().unwrap().recv().unwrap();

                println!("Worker {} got a job; executing.", id);

                job.call_box();
            }
        });

        Worker {
            id,
            thread,
        }
    }
}
```

## Graceful Shutdown 与清理

如果我们编译上述表示的代码，会有一些警告说存在一些字段并没有直接被使用，这提醒了我们并没有清理任何内容。当使用 `ctrl-C` 终止主线程，所有其他线程也会立刻停止，即便他们正在处理一个请求。现在我们要为 `ThreadPool` 实现 `Drop trait` 对线程池中的每一个线程调用 `join`，这样这些线程将会执行完他们的请求。接着会为 `ThreadPool` 实现一个方法来告诉线程他们应该停止接收新请求并结束。为了实践这些代码，修改 `server` 在 `graceful Shutdown` 之前只接受两个请求。

```
enum Message {
    NewJob(Job),
    Terminate,
}

pub struct ThreadPool {
    workers: Vec<Worker>,
    sender: mpsc::Sender<Message>,
}

// ...snip...

impl ThreadPool {
    // ...snip...
    pub fn new(size: usize) -> ThreadPool {
        assert!(size > 0);

        let (sender, receiver) = mpsc::channel();

        // ...snip...
    }

    pub fn execute<F>(&self, f: F)
        where
            F: FnOnce() + Send + 'static
    {
        let job = Box::new(f);

        self.sender.send(Message::NewJob(job)).unwrap();
    }
}

// ...snip...

impl Worker {
    fn new(id: usize, receiver: Arc<Mutex<mpsc::Receiver<Message>>>) ->
        Worker {

        let thread = thread::spawn(move ||{
            loop {
                let message = receiver.lock().unwrap().recv().unwrap();

                match message {
                    Message::NewJob(job) => {
                        println!("Worker {} got a job; executing.", id);

                        job.call_box();
                    },
                    Message::Terminate => {
                        println!("Worker {} was told to terminate.", id);

                        break;
                    },
                }
            }
        });

        Worker {
            id,
            thread: Some(thread),
        }
    }
}
```

收发 `Message` 值并在 Worker 收到 `Message::Terminate` 时退出循环

需要将 `ThreadPool` 定义、创建通道的 `ThreadPool::new` 和 `Worker::new` 签名中的 `Job` 改为 `Message`。`ThreadPool` 的 `execute` 方法需要发送封装进 `Message::NewJob` 成员的任务，当获取到 `NewJob` 时会处理任务而收到 `Terminate` 成员时则会退出循环。

```
impl Drop for ThreadPool {
    fn drop(&mut self) {
        println!("Sending terminate message to all workers.");

        for _ in &mut self.workers {
            self.sender.send(Message::Terminate).unwrap();
        }

        println!("Shutting down all workers.");

        for worker in &mut self.workers {
            println!("Shutting down worker {}", worker.id);

            if let Some(thread) = worker.thread.take() {
                thread.join().unwrap();
            }
        }
    }
}
```

在对每个 `worker` 线程调用 `join` 之前向 `worker` 发送 `Message::Terminate`

现在遍历了 `worker` 两次，一次向每个 `worker` 发送一个 `Terminate` 消息，一个调用每个 `worker` 线程上的 `join`。如果尝试在同一循环中发送消息并立即 `join` 线程，则无法保证当前迭代的 `worker` 是从通道收到终止消息的` worker`。

为了更好的理解为什么需要两个分开的循环，想象一下只有两个 `worker` 的场景。如果在一个循环中遍历每个`worker`，在第一次迭代中 `worker` 是第一个 `worker`，我们向通道发出终止消息并对第一个 `worker` 线程调用 `join`。如果第一个 `worker` 当时正忙于处理请求，则第二个 `worker` 会从通道接收这个终止消息并结束。而我们在等待第一个 `worker` 结束，不过它永远也不会结束因为第二个线程取走了终止消息。现在我们就阻塞在了等待第一个 `worker` 结束，而无法发出第二条终止消息。死锁！

为了避免此情况，首先从通道中取出所有的 `Terminate` 消息，接着 `join` 所有的线程。因为每个 `worker` 一旦收到终止消息即会停止从通道接收消息，我们就可以确保如果发送同 `worker` 数相同的终止消息，在 `join` 之前每个线程都会收到一个终止消息。

#### 简易线程池完整代码

```
// lib.rs
use std::thread;
use std::sync::mpsc;
use std::sync::Arc;
use std::sync::Mutex;

enum Message {
    NewJob(Job),
    Terminate,
}

pub struct ThreadPool{
    workers: Vec<Worker>,
    sender: mpsc::Sender<Message>,
}

type Job = Box<FnBox + Send + 'static>;

impl ThreadPool {
    /// Create a new ThreadPool.
    ///
    /// The size is the number of threads in the pool.
    ///
    /// # Panics
    ///
    /// The `new` function will panic if the size is zero.
    pub fn new(size :usize) ->ThreadPool{
        assert!(size > 0);

        let (sender, receiver) = mpsc::channel();

        let receiver = Arc::new(Mutex::new(receiver));

        let mut workers = Vec::with_capacity(size);

        for id in 0..size {
            workers.push(Worker::new(id, receiver.clone()));
        }

        ThreadPool {
            workers,
            sender,
        }
    }

    pub fn execute<F> (&self, f: F)
        where
        F: FnOnce() + Send + 'static
    {
        let job = Box::new(f);
        self.sender.send(Message::NewJob(job)).unwrap();
    }
}

impl Drop for ThreadPool {
    fn drop(&mut self) {
        println!("Sending terminate message to all workers.");

        for _ in &mut self.workers {
            self.sender.send(Message::Terminate).unwrap();
        }

        println!("Shutting down all workers.");

        for worker in &mut self.workers {
            println!("Shutting down worker {}", worker.id);

            if let Some(thread) = worker.thread.take() {
                thread.join().unwrap();
            }
        }
    }
}

trait FnBox {
    fn call_box(self: Box<Self>);
}

impl<F: FnOnce()> FnBox for F {
    fn call_box(self: Box<F>) {
        (*self)()
    }
}

struct Worker {
    id: usize,
    thread: Option<thread::JoinHandle<()>>,
}

impl Worker {
    fn new(id: usize, receiver: Arc<Mutex<mpsc::Receiver<Message>>>) -> Worker {
        let thread = thread::spawn(move || {
            loop {
                let message = receiver.lock().unwrap().recv().unwrap();

                match message {
                    Message::NewJob(job) => {
                        println!("Worker {} got a job; executing.", id);

                        job.call_box();
                    },
                    Message::Terminate => {
                        println!("Worker {} was told to terminate.", id);

                        break;
                    },
                }
            }
        });

        Worker {
            id,
            thread: Some(thread),
        }
    }
}
```

## 参考

1. [The Rust Programming Language](https://doc.rust-lang.org/book/second-edition/ch20-00-final-project-a-web-server.html)
2. [Rust程序设计-中文版](https://kaisery.github.io/trpl-zh-cn/)
