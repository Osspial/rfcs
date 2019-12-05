- Feature Name: overloadable-future-context
- Start Date: (fill me in with today's date, YYYY-MM-DD)
- RFC PR: [rust-lang/rfcs#0000](https://github.com/rust-lang/rfcs/pull/0000)
- Rust Issue: [rust-lang/rust#0000](https://github.com/rust-lang/rust/issues/0000)

# Summary
[summary]: #summary

As of Version 1.39, Rust has a mechanism for constructing arbitrarily complex
state machines in a clear an intuitive way built directly into the language via
`async`/`await`. However, it is only easily usable in the context of a
high-level task system. This RFC proposes a relatively small expansion to
`async`/`await` and the associated `Future` trait that generalizes that
mechanism, unleashing its power throughout the rest of the language, by
redefining `std::future::Future` as follows:

```rust
pub trait Future {
    type Output;
    type Context<'a> = &'a mut std::task::Context;
    fn poll(self: Pin<&mut Self>, cx: Self::Context<'_>) -> Poll<Self::Output>;
}
```

# Motivation
[motivation]: #motivation


* It is sometimes useful to run `Future`s outside the context of an executor.
    - Example: generators and coroutines
* Allowing users to re-define the `Context` type will allow users to implement generators and related features on top of the `async/await` API, without having to define new language features.
* It isn't always desirable for `Future`s to be completely decoupled from their executors.
    - There are situations where futures need to have precise control over the conditions of their execution in order to provide an acceptable user experience.
    - Example: graphical application event loops.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

## Context: `Waker`s and the `Task` API

https://boats.gitlab.io/blog/post/wakers-i/

It's perfectly valid to use use `async`/`await` as a lightweight task system. However,
it's a mistake to view `Future`s *exclusively* as tasks; instead, `async` functions
are a method of transforming imperative blocks into arbitrary state machines. By providing
a custom `Context` type, you can use the `async` API outside of the standard task system.

## Case study: generators

```rust
//! Implementing a fibonacci sequence generator with iterators.
struct Fibonacci {
    current: u32,
    next: u32,
}

impl Fibonacci {
    fn new() -> Fibonacci {
        Fibonacci {
            current: 0,
            next: 1,
        }
    }
}

impl Iterator for Fibonacci {
    type Item = u32;
    fn next(&mut self) -> Option<u32> {
        let ret = self.current;

        let new = self.current + self.next;
        self.current = self.next;
        self.next = new;
        Some(ret)
    }
}

fn main() {
    for i in Fibonacci::new().take(1000) {
        println!("{:?}", i);
    }
}
```

```rust
//! Defining a future-based generator API.
pub struct Generator<T, F>
    where for<'a> F: Future<Output=(), Context<'a>=&mut Option<T>>,
{
    future: F,
    _t: PhantomData<T>,
}

pub struct Yield<T> {
    value: Option<T>,
}

pub fn r#yield(value: T) -> Yield<T> {
    Yield {
        value: Some(value),
    }
}

impl<T> Future for Yield<T> {
    type Output = ();
    type Context<'a> = &'a mut Option<T>;
    fn poll(self: Pin<&mut Self>, cx: &mut Option<T>) -> Poll<()> {
        cx.value = self.value.take();
        Poll::Ready(())
    }
}

impl<T, F> Iterator for Generator<T, F>
    where for<'a> F: Future<Output=(), Context<'a>=&mut Option<T>>,
{
    type Item = T;
    fn next(&mut self) -> Option<T> {
        let mut value = None;
        match self.future.poll(&mut value) {
            Poll::Pending => value,
            Poll::Ready(()) => None,
        }
    }
}

impl<T, F> Generator<T, F>
    where for<'a> F: Future<Output=(), Context<'a>=&mut Option<T>>,
{
    pub fn new(future: F) -> Generator<T, F> {
        Generator {
            future,
            value: None,
        }
    }
}

fn main() {
    let fibonacci = Generator::new(async {
        let mut current = 0;
        let mut next = 1;
        loop {
            r#yield(current).await;

            let new = next + current;
            current = next;
            next = new;
        }
    });

    for i in fibonacci.take(10) {
        println!("{:?}", i);
    }
}
```

## Case study: simple event loops

This section demonstrates how overloadable contexts allow users to use `async` to
create complex state machines.

```rust
//! Based on a heavily simplified version of Winit's event loop.

fn main() {
    let event_loop = EventLoop::new();
    let window = Window::new(&event_loop);

    event_loop.run(|event, control_flow| {
        match event {
            Event::NewEvents => println!("event loop iteration begun"),
            Event::WindowEvent(event) => match event {
                WindowEvent::CharReceived('x') => *control_flow = ControlFlow::Exit,
                WindowEvent::CharReceived('p') => *control_flow = ControlFlow::Poll,
                WindowEvent::CharReceived('w') => *control_flow = ControlFlow::Wait,
                WindowEvent::CharReceived(c) => println!("char received: {:?}", c),
                _ => (),
            },
            Event::EventsCleared => {
                // render to window
                println!("event loop iteration completed");
            },
            Event::LoopDestroyed => println!("event loop shutting down"),
        }
    });
}

#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub enum WindowEvent {
    CharReceived(char),
    /*..*/
}
#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub enum Event {
    NewEvents,
    WindowEvent(WindowEvent),
    EventsCleared,
    LoopDestroyed,
}

pub enum ControlFlow {
    Wait,
    Poll,
    Exit,
}

pub struct EventLoop {/*..*/}
impl EventLoop {
    pub fn new() -> EventLoop {/*..*/}
    pub fn run(self, event_handler: impl FnMut(Event, &mut ControlFlow)) {/*..*/}
}

pub struct Window {/*..*/}
impl Window {
    pub fn new(event_loop: &EventLoop) -> Window {/*..*/}
}
```

```rust
fn main() {
    let event_loop = EventLoop::new();
    let window = Window::new(&event_loop);

    event_loop.run_async(async {
        let mut wait_for_events = false;

        'event_loop: loop {
            if wait_for_events {
                wait_for_events().await;
            }

            println!("event loop iteration begun");

            let mut event_receiver = EventReceiver::new().await;
            while let Some(window_event) = event_receiver.next_event().await {
                match window_event {
                    WindowEvent::CharReceived('x') => break 'event_loop,
                    WindowEvent::CharReceived('p') => wait_for_events = false,
                    WindowEvent::CharReceived('w') => wait_for_events = true,
                    WindowEvent::CharReceived(c) => println!("char received: {:?}", c),
                    _ => (),
                }
            }

            // render to window
            println!("event loop iteration completed");
        }

        println!("event loop shutting down");
    });
}


pub struct EventLoopContext<'a> {
    event: &'a mut Option<Event>,
    control_flow: &'a mut ControlFlow,
}

impl EventLoop {
    pub fn run_async(self, event_handler: impl for<'a> Future<Output=(), Context<'a>=EventLoopContext<'a>>) {
        let mut pinned_event_handler = Some(Box::new(event_handler).pin());
        event_loop.run(|event, control_flow| {
            let mut event_opt = Some(event);
            let context = EventLoopContext {
                event: &mut event_opt,
                control_flow,
            };

            while context.event.is_some() {
                let poll_result = pinned_event_handler
                    .as_mut()
                    .map(|handler| handler.poll(context))
                    .unwrap_or(Poll::Ready(()));

                match poll_result {
                    Poll::Pending => (),
                    Poll::Ready(()) => {
                        *control_flow = ControlFlow::Exit;
                        break;
                    },
                }
            }
        });
    }
}

pub fn wait_for_events() -> impl for<'a> Future<Output=(), Context<'a>=EventLoopContext<'a>> {
    struct WaitForEvents;
    impl Future for WaitForEvents {
        type Output = ();
        type Context<'a> = EventLoopContext<'a>;
        fn poll(self: Pin<&mut Self>, cx: EventLoopContext<'a>) -> Poll<()> {
            match *cx.event {
                None => Poll::Pending,
                Some(Event::WindowEvent(_)) |
                Some(Event::EventsCleared) => {
                    *cs.event = None;
                    *cs.control_flow = ControlFlow::Wait;
                    Poll::Pending
                },
                Some(Event::NewEvents) => {
                    *cs.control_flow = ControlFlow::Poll;
                    Poll::Ready(())
                }
            }
        }
    }
    WaitForEvents
}

pub struct EventReceiver {
    _foo: (),
}

impl EventReceiver {
    pub fn new() -> impl for<'a> Future<Output=EventReceiver, Context<'a>=EventLoopContext<'a>> {
        struct EventReceiverBuilder;
        impl Future for EventReceiverBuilder {
            type Output = EventReceiver;
            type Context<'a> = EventLoopContext<'a>;
            fn poll(self: Pin<&mut Self>, cx: EventLoopContext<'a>) -> Poll<EventReceiver> {
                match cx.event.take() {
                    None |
                    Some(Event::WindowEvent(_)) |
                    Some(Event::EventsCleared) => {
                        Poll::Pending
                    },
                    Some(Event::NewEvents) => {
                        Poll::Ready(EventReceiver {_foo: ()})
                    },
                }
            }
        }
        EventReceiverBuilder
    }

    pub fn next_event<'b>(&'b mut self)
        -> impl for<'a> 'b + Future<Output=Option<WindowEvent>, Context<'a>=EventLoopContext<'a>>
    {
        struct EventReceiverFuture;
        impl Future for EventReceiverFuture {
            type Output = Option<WindowEvent>;
            type Context<'a> = EventLoopContext<'a>;
            fn poll(self: Pin<&mut Self>, cx: EventLoopContext<'a>) -> Poll<Option<WindowEvent>> {
                match cx.event {
                    None => Poll::Pending,
                    Some(Event::WindowEvent(event)) => {
                        *cx.event = None;
                        Poll::Ready(Some(event))
                    },
                    Some(Event::EventsCleared) => Poll::Ready(None),
                    Some(Event::NewEvents) => unreachable!(),
                }
            }
        }

        EventReceiverFuture
    }
}
```

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

## `async` blocks



This is the technical portion of the RFC. Explain the design in sufficient detail that:

- Its interaction with other features is clear.
- It is reasonably clear how the feature would be implemented.
- Corner cases are dissected by example.

The section should return to the examples given in the previous section, and explain more fully how the detailed proposal makes those examples work.

# Drawbacks
[drawbacks]: #drawbacks

Why should we *not* do this?

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

- Why is this design the best in the space of possible designs?
- What other designs have been considered and what is the rationale for not choosing them?
- What is the impact of not doing this?

# Prior art
[prior-art]: #prior-art

Discuss prior art, both the good and the bad, in relation to this proposal.
A few examples of what this can include are:

- For language, library, cargo, tools, and compiler proposals: Does this feature exist in other programming languages and what experience have their community had?
- For community proposals: Is this done by some other community and what were their experiences with it?
- For other teams: What lessons can we learn from what other communities have done here?
- Papers: Are there any published papers or great posts that discuss this? If you have some relevant papers to refer to, this can serve as a more detailed theoretical background.

This section is intended to encourage you as an author to think about the lessons from other languages, provide readers of your RFC with a fuller picture.
If there is no prior art, that is fine - your ideas are interesting to us whether they are brand new or if it is an adaptation from other languages.

Note that while precedent set by other languages is some motivation, it does not on its own motivate an RFC.
Please also take into consideration that rust sometimes intentionally diverges from common language features.

# Unresolved questions
[unresolved-questions]: #unresolved-questions

- What parts of the design do you expect to resolve through the RFC process before this gets merged?
- What parts of the design do you expect to resolve through the implementation of this feature before stabilization?
- What related issues do you consider out of scope for this RFC that could be addressed in the future independently of the solution that comes out of this RFC?

# Future possibilities
[future-possibilities]: #future-possibilities

Think about what the natural extension and evolution of your proposal would
be and how it would affect the language and project as a whole in a holistic
way. Try to use this section as a tool to more fully consider all possible
interactions with the project and language in your proposal.
Also consider how the this all fits into the roadmap for the project
and of the relevant sub-team.

This is also a good place to "dump ideas", if they are out of scope for the
RFC you are writing but otherwise related.

If you have tried and cannot think of any future possibilities,
you may simply state that you cannot think of anything.

Note that having something written down in the future-possibilities section
is not a reason to accept the current or a future RFC; such notes should be
in the section on motivation or rationale in this or subsequent RFCs.
The section merely provides additional information.
