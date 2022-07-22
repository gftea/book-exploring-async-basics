# Implementing our own runtime

Let's start to get some code written down; we have a lot to do.

The way we'll go about this is that I'll go through everything the way I find it easiest to parse and understand. That also means that sometimes I have to introduce a bit of code that will be explained later.  Try not to worry if you don't understand something at first.  I'll be going through everything.

The very first thing we need to do is to create a Rust project to run our code in:

```
cargo new async-basics
cd async-basics
```

Now, as I've explained, we'll need to use the `minimio` library.

> You can find a thorough explanation of minimio in the book: [Epoll, Kqueue and IOCP Explained with Rust](https://cfsamsonbooks.gitbook.io/epoll-kqueue-iocp-explained/). The [second part](https://cfsamsonbooks.gitbook.io/epoll-kqueue-iocp-explained/the-recipie-for-an-eventqueue) is all about minimio.

In `Cargo.toml`

```toml
[dependencies]
minimio = {git = "https://github.com/cfsamson/examples-minimio", branch = "master"}
```

A second option is to clone the repository containing all the code we're going
to write and go through that as we go along:

```
git clone https://github.com/cfsamson/examples-node-eventloop
```

The next thing we need is a `Runtime` to hold all the state our `Runtime` needs.

First navigate to `main.rs` (located in `src/main.rs`).

> We'll write everything in one file this time in roughly the same order as we
> go through them in this book.

## Runtime struct

I've added comments to the code so it's easier to remember and understand.

```rust, ignore
pub struct Runtime {
    /// Available threads for the threadpool
    available_threads: Vec<usize>,
    /// Callbacks scheduled to run
    callbacks_to_run: Vec<(usize, Js)>,
    /// All registered callbacks
    callback_queue: HashMap<usize, Box<dyn FnOnce(Js)>>,
    /// Number of pending epoll events, only used by us to print for this example
    epoll_pending_events: usize,
    /// Our event registrator which registers interest in events with the OS
    epoll_registrator: minimio::Registrator,
    // The handle to our epoll thread
    epoll_thread: thread::JoinHandle<()>,
    /// None = infinite, Some(n) = timeout in n ms, Some(0) = immediate
    epoll_timeout: Arc<Mutex<Option<i32>>>,
    /// Channel used by both our threadpool and our epoll thread to send events
    /// to the main loop
    event_receiver: Receiver<PollEvent>,
    /// Creates an unique identity for our callbacks
    identity_token: usize,
    /// The number of events pending. When this is zero, we're done
    pending_events: usize,
    /// Handles to our threads in the threadpool
    thread_pool: Vec<NodeThread>,
    /// Holds all our timers, and an Id for the callback to run once they expire
    timers: BTreeMap<Instant, usize>,
    /// A struct to temporarely hold timers to remove. We let Runtinme have
    /// ownership so we can reuse the same memory
    timers_to_remove: Vec<Instant>,
}
```

Now, I've added some comments here to explain what they're for and in the coming
chapters we'll cover every one of them.

I'll continue by defining some of the types we use here.

## Task

```rust, ignore
struct Task {
    task: Box<dyn Fn() -> Js + Send + 'static>,
    callback_id: usize,
    kind: ThreadPoolTaskKind,
}

impl Task {
    fn close() -> Self {
        Task {
            task: Box::new(|| Js::Undefined),
            callback_id: 0,
            kind: ThreadPoolTaskKind::Close,
        }
    }
}
```

We need a task object, which represents a task we want to finish in our thread
pool. I'll go through the types in this object in a later [chapter](./8_9_infrastructure.md) so don't worry too much about them now if you find them
hard to grasp. Everything will be explained.

We also create an implementation of a `Close` task. We need this to clean up after ourselves and close down the thread pool.

`|| Js::Undefined` might seem strange but it's only a function that returns `Js::Undefined`, we need it since we won't make `task` an `Option` just for this one case.

It's just so we don't have to `match` or `map` on `task` all the way through our code, it's more than enough to parse already.

## NodeThread

First is `NodeThread`, which represents a thread in our thread pool. As you
see we have a `JoinHandle` (which we get when we call `thread::spawn`) and the
sending part of a channel. This channel, sends messages of the type `Task`.

```rust, ignore
#[derive(Debug)]
struct NodeThread {
    pub(crate) handle: JoinHandle<()>,
    sender: Sender<Task>,
}
```

We introduced two new types here: `Js` and `ThreadPoolTaskKind`. First we'll cover `ThreadPoolTaskKind`.

In our example, we have three kinds of events: a `FileRead` which is a file that has been read, and an `Encrypt` that represents an operation from our `Crypto` module. The event `Close` is used to tell the threads in our `threadpool` that we're closing the loop and let them finish before we exit our process.

## ThreadPoolTaskKind

As you might understand, this object is only used in the `threadpool`.

```rust, ignore
pub enum ThreadPoolTaskKind {
    FileRead,
    Encrypt,
    Close,
}
```

## Js

Next is our `Js` object. This represents different `Javascript` types, and it's only used
to make our code look more "javascripty", but it's also convenient for us to
abstract over the return types of closures.

We'll also implement two convenience methods on this object to make our "javascripty"
code look a bit cleaner.

We know the return types already based on our modules
documentation - just like you would know it from the documentation when using a
Node module but we need to actually handle the types in Rust so this will make that just slightly easier for us.

```rust, ignore
#[derive(Debug)]
pub enum Js {
    Undefined,
    String(String),
    Int(usize),
}

impl Js {
    /// Convenience method since we know the types
    fn into_string(self) -> Option<String> {
        match self {
            Js::String(s) => Some(s),
            _ => None,
        }
    }

    /// Convenience method since we know the types
    fn into_int(self) -> Option<usize> {
        match self {
            Js::Int(n) => Some(n),
            _ => None,
        }
    }
}
```

## PollEvent

Next we have the `PollEvent`. While we defined an `enum` to represent what
kind of events we could send **to** the `eventpool`, we define some events
that we can accept back from both our `epoll based` event queue and our `threadpool`.

```rust, ignore
/// Describes the three main events our epoll-eventloop handles
enum PollEvent {
    /// An event from the `threadpool` with a tuple containing the `thread id`,
    /// the `callback_id` and the data which the we expect to process in our
    /// callback
    Threadpool((usize, usize, Js)),
    /// An event from the epoll-based eventloop holding the `event_id` for the
    /// event
    Epoll(usize),
    Timeout,
}
```

## const RUNTIME

Lastly we have another convenience for us, and it's also necessary to make our
Javascript code look a bit like Javascript.

First we have a static variable which represents our `Runtime`. It's actually
a pointer to our `runtime` which we initialize to a null-pointer from the start.

We need to use unsafe to edit this. I'll explain later how this is safe, but I
also want to mention here that it could be avoided by using [lazy_static](https://github.com/rust-lang-nursery/lazy-static.rs)
but that would both require us to add `lazy_static` as an dependency (which would
be fine since it contains no "magic" that we want to explain in this book) but
it also does make our code less readable, and it's complex enough.


```rust, ignore
static mut RUNTIME: *mut Runtime = std::ptr::null_mut();
```

Let's now move on and look at what the heart of the runtime looks like: the main loop

## Moving on

Now we've already gotten really far by introducing most of our runtime already in the first chapter. The next chapter will focus on implementing all the functionality we need for this to work.
