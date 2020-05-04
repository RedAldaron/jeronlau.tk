+++
title="Reinventing Asynchronous Rust"
date=2020-05-04
+++

## A Brief Intro To Asynchronous Programming In Rust
Lets say you want to write some `async`/`.await` code in Rust, since it's new
and cool.  Let's start with the simplest async program that you could write (a
timer).  First we have to create a `Future`, something that completes at some
point in the future but won't start unless you execute it (in Rust, this is
called lazy evaluation).  Here's some code to start:

<!-- more -->

```rust
// Sleep for 1 second
fn blocking_timer() {
    std::thread::sleep(std::time::Duration::from_millis(1000));
}

fn main() {
    let thread = std::thread::spawn(blocking_timer);
    // TODO: Turn the thread into a Future
    thread.await; // TODO: Doesn't work
}
```

So we have two problems here, first of all we can't `.await` something unless
it's inside of an `async fn` (function that returns a `Future`, rather than
executing when called).  And second, how do we make a `Future` out of our
`thread`?  Turns out the Rust standard library does not have an executor built
in, so we have to pick one.  Three popular async executors are the `futures`
crate, `async-std` and `tokio`.  `async-std` and `tokio` have other stuff for
actually doing I/O built in that we don't need, so we'll go with `futures`.  So,
it Cargo.toml add

```toml
[dependencies]
futures = "0.3"
```

and then do `cargo build`.  27 crates get compiled (wow, that's a lot!  Just to
be able to use a language feature, but could be worse).  Looks like futures has
everything we need to finish our example, though.  And, a lot more (it's kind of
overwhelming IMO).

```rust
// Sleep for 1 second
fn blocking_timer(sender: futures::channel::oneshot::Sender<()>) {
    std::thread::sleep(std::time::Duration::from_millis(1000));

    sender.send(()).unwrap();
}

fn main() {
    let (sender, receiver) = futures::channel::oneshot::channel::<()>();
    let _thread = std::thread::spawn(move || blocking_timer(sender));

    // Turn the thread into a Future
    let thread = async { receiver.await };

    // Execute the future
    futures::executor::block_on(thread).unwrap();
}
```

Yay, we made a future and executed it!  Now I don't think this API is too
terrible, but it would be nice if I didn't have to worry about handling the
`Results` with unwraps.  Also, the types are buried deep in a module tree which
made them very hard to find for me (3 levels deep!!).  I think I can do better.

## Stuff Gets Gross
It gets worse than a few minor complaints.  Note this code from the `Future`s
documentation.

```rust
use futures::future::FutureExt;
use futures::select;
use futures::pin_mut;

// Calling the following async fn returns a Future which does not
// implement Unpin
async fn async_identity_fn(arg: usize) -> usize {
    arg
}

let fut_1 = async_identity_fn(1).fuse();
let fut_2 = async_identity_fn(2).fuse();
let mut fut_1 = Box::pin(fut_1); // Pins the Future on the heap
pin_mut!(fut_2); // Pins the Future on the stack

let res = select! {
    a_res = fut_1 => a_res,
    b_res = fut_2 => b_res,
};
assert!(res == 1 || res == 2);
```

We have to pin `fut_2` because of the nature of this API, but unfortunately
we're using the `pin_mut!()` macro, which is problematic.  Try adding
`#![forbid(unsafe_code)]` to the top of your file and see what happens.  Guess
what, it doesn't compile even though you didn't write any unsafe code!  Yes, the
macro inserts `unsafe` blocks into your code (gross).  If you're writing an
application in Rust, and you don't need raw FFI calls in your code, it should be
`#![forbid(unsafe_code)]`, and this makes that impossible!  When I discovered
this, I decided to make a replacement crate for `futures`.

If you thought that was gross, guess what happens when you try and mix and match
crates that depend on `async-std` and `tokio`?  Well, it doesn't work because of
some incompatable async runtime nonsense.  And now the async ecosystem is
splitting (oh no!).

Now, look at the above code and tell me that the `select!` macro isn't ugly.
No, it's just gross to look at and read.  And why do I even have to worry about
pinning?  I just want to execute a `Future`!

## Don't Give Up Hope
I have found a solution to every problem with async in Rust (Well, at least the
ones I mentioned).  And, I want to share it with you.  That solution is
[https://crates.io/crates/pasts](https://crates.io/crates/pasts), a minimal and simpler alternative to the futures crate.
This solution doesn't use any macros at all, but is instead based on traits, and
as a bonus has zero dependencies and works in a `no-std` environment.
Additionally, I believe there's nothing in it that prevents it from being usable
with `tokio` or `async-std` (futures from those crates should be able to run
within the pasts executor).  Here's the example above rewritten using
`pasts 0.1.0`, released yesterday:

```rust
use pasts::prelude::*;

fn main() {
    // spawn_blocking returns a Future that returns Ready when the thread exits.
    pasts::ThreadInterrupt::block_on(pasts::spawn_blocking(|| {
        std::thread::sleep(std::time::Duration::from_millis(1000))
    }));
}
```

And the select code example from the `futures` crate documentation rewritten:

```rust
use pasts::prelude::*;

// Calling the following async fn returns a Future which does not
// implement Unpin
async fn async_identity_fn(arg: usize) -> usize {
    arg
}

let res = [async_identity_fn(1).fut(), async_identity_fn(2).fut()]
    .select()
    .await
    .1;

assert!(res == 1 || res == 2);
```

As you can see, the code (IMO) is a lot cleaner and easier to follow.  I hope
that this library will help Rustaceans not have to worry about building their
own async runtime because a "general purpose runtime" is useless (according to
some other Rust blog posts).  This library doesn't spawn any threads outside of
the `spawn_blocking` API, so you get to choose which threads a `Future` runs on
at compile time.  I personally think this approach is as close as you can get to
an async library that's general purpose.  This also means you can use `RefCell`
to share state between futures instead of a specialized `Mutex` type, as some
async libraries provide.

## That's Not All!
Now pasts is great for the application side of `async`/`.await`, but what about
the library side?  If you're making an async library, you're probably providing
your own `Future`s, which is trickier to do than it sounds.  You need to have a
mechanism to wake your futures, and thus comes
[smelling_salts](https://crates.io/crates/smelling_salts) 0.1.0 (also released
yesterday)!

Say we don't want to start a new thread every time we want a timer future,
because we'll be doing it a lot.  Instead we'll use a native Linux API, timerfd:

```rust
use smelling_salts::{Device, Watcher};
use std::os::unix::io::RawFd;
use std::os::raw::{c_long, c_int, c_void};
use std::task::{Context, Poll};
use std::pin::Pin;
use std::future::Future;
use std::mem::MaybeUninit;

#[repr(C)]
struct TimeSpec {
    // struct timespec, from C.
    tv_sec: c_long,
    tv_nsec: c_long,
}

#[repr(C)]
struct ItimerSpec {
    // struct itimerspec, from C.
    it_interval: TimeSpec,
    it_value: TimeSpec,
}

extern "C" {
    fn timerfd_create(clockid: c_int, flags: c_int) -> RawFd;
    fn timerfd_settime(
        fd: RawFd,
        flags: c_int,
        new_value: *const ItimerSpec,
        old_value: *mut ItimerSpec,
    ) -> c_int;
    fn read(fd: RawFd, buf: *mut c_void, count: usize) -> isize;
    fn close(fd: RawFd) -> c_int;
}

/// A timer Future that completes after 1 second.
pub struct SecondTimer {
    device: Device,
}

impl SecondTimer {
    /// Create a new timer that will expire in 1 second.
    pub fn new() -> Self {
        // Create and set timer to repeat every second, starting in 1 second.
        let timerfd = unsafe {
            timerfd_create(
                1,      /*CLOCK_MONOTONIC*/
                0o4000, /*TFD_NONBLOCK*/
            )
        };
        assert_ne!(timerfd, -1);
        unsafe {
            timerfd_settime(
                timerfd,
                0,
                &ItimerSpec {
                    it_interval: TimeSpec {
                        tv_sec: 1,
                        tv_nsec: 0,
                    },
                    it_value: TimeSpec {
                        tv_sec: 1,
                        tv_nsec: 0,
                    },
                },
                std::ptr::null_mut(),
            );
        }
        // Create timer device, watching for input events.
        let device = Device::new(timerfd, Watcher::new().input());
        
        SecondTimer { device }
    }
}

impl Future for SecondTimer {
    type Output = ();
    
    fn poll(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Self::Output> {
        let mut num = MaybeUninit::<u64>::uninit();
        if unsafe {
            read(
                self.device.fd(),
                num.as_mut_ptr().cast(),
                std::mem::size_of::<u64>(),
            )
        } == std::mem::size_of::<u64>() as isize {
            // Read succeeded, future completed.
            Poll::Ready(())
        } else {
            // Watch device and wake when new data is ready to be read.
            self.device.register_waker(cx.waker());
            // Put future to sleep.
            Poll::Pending
        }
    }
}

impl Drop for SecondTimer {
    fn drop(&mut self) {
        let fd = self.device.fd();
        self.device.old();
        assert_ne!(unsafe { close(fd) }, -1);
    }
}
```

Libraries should not depend on pasts, because they don't need to.  This `Future`
will run on any executor (async-std, tokio, pasts, others), and it should be the
application author's decision which one to use (for maximum compatability).

Now, actually using the future in your application with pasts:

```rust
use pasts::prelude::*;
use library_name::SecondTimer;

fn main() {
    pasts::ThreadInterrupt::block_on(SecondTimer::new());
}
```

## Conclusion
I think my unique approach to handling futures in Rust solves a lot of common
problems, and hopefully you find it useful, too.  If you want to see it in
action, I released async versions of my crates `wavy` and `stick` yesterday as
well (links below).

### Links
- [pasts](https://crates.io/crates/pasts) - A minimal and simpler alternative to
  the `futures` crate.
- [smelling_salts](https://crates.io/crates/smelling_salts) - Wake futures using
  an epoll thread without depending on an executor / runtime.
- [stick](https://crates.io/crates/stick) - A library that provides futures for
  gamepads, joysticks and other controllers (currently only Linux).
- [wavy](https://crates.io/crates/wavy) - A library that provides futures for
  doing real time audio input and output (currently only Linux).

> *Feel free to email me any corrections in grammar, spelling or technical errors at [jeronlau@plopgrizzly.com](mailto:jeronlau@plopgrizzly.com).*