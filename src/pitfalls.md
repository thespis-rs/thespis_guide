# Pitfalls Checklist

The actor model is a very convenient programming paradigm. It automatically solves many of the harder problems in concurrent software design. However, there are a few pitfalls, so here is a check list of things the compiler won't catch for us:


## 1.Deadlocks

The original actor model is very simple. If you want to have a request-response type message, you had to send your own address along with the message so the recipient knows how to respond to you. Thanks to Rust's futures, we can easily create a request-response and _thespis_ makes this convenient for you with `Address::call`. However, if you await a response whilst processing a message, a deadlock can arise.

As actors only process one message at a time. If you have a cyclic dependency, your program can deadlock. That is if actor A depends on actor B to to process it's message, but in order to process A's request B also needs to call back A, both actors will deadlock, as they are waiting on each other and will no longer process any messages ever after.

You can run into this problem through intermediate actors, so A and B might not be calling each other directly.

You can also run into this problem with a single actor when using bounded channels. If the mailbox of the actor is currently full and it sends itself a message, it will deadlock. You should generally spawn such a send to avoid blocking processing of the current message.

Another deadlock issue can arise when an actor is the gatekeeper of a connection (to the network or other components). That is if the mailbox of one actor accumulates both incoming and outgoing messages. If incoming messages saturate the system, and fill up the gatekeeper's mailbox, no responses can flow out. Hence no place is made for new incoming messages and everything blocks. You might want to read up on this issue [in this article](https://elizarov.medium.com/deadlocks-in-non-hierarchical-csp-e5910d137cc) which has some nice diagrams to explain the problem and [on the trio forum](https://trio.discourse.group/t/sizing-the-channel-deadlock-freedom-vs-back-pressure). This problem can be solved by not using the channels for backpressure, or by using a priority channel. _Thespis_impl_ has [an example](https://github.com/thespis-rs/thespis_impl/blob/dev/examples/deadlock_prio.rs) demonstrating this last solution.


## 2. Memory consumption

The actor model has highly concurrent message passing. If many messages are in flight, they will all consume memory. The most obvious risk is having unbounded channels to communicate between address and mailbox. If there is no back pressure, the unbounded channel can use unbounded amounts of memory.

But even with bounded channels you want to keep an eye on memory consumption. If you spawn large volumes of actors, you probably want to use a channel that doesn't allocate it's full capacity. Channels based on linked lists rather than array buffers could potentially reduce a lot of memory consumption.


## 3. Performance

The main downside of the actor model is a performance overhead compared to lower level approaches. Message passing has some price and depending on your application it might be a poor fit. If you have hotspots in your code, and you need to do a lot of small operations, like calling getters on other objects, the overhead of message passing might be undesirable.

Remember that the purpose of this model is to have an actor for everything that you want to run concurrently. However, you can have a module for which an actor is the facade to the outside world, but internally uses a synchronous logic. You do however want to reach a critical threshold of actors. If you have CPU cores x 4 components that run concurrently and that normally are rarely blocked, eg. most of them will be able to make progress most of the time, adding more concurrency to your program might not be the most useful for performance.

That being said, you shouldn't worry about it to much. Rust is a very performant language. Even with the overhead of message passing, it will be much faster than most other languages, and remember how much useful software has been written in those languages. In 99% of the time performance shouldn't be an issue.

As always, first write your software with clean design, then benchmark to see if there is a problem. If there is, profile to see what's your bottleneck.


## 4. Blocking

We all know that async code shouldn't block. That is we shouldn't block the thread with either to much CPU intensive work, or by waiting on things like locks. That's because other tasks might also be working on the same thread.

However it's important to also think about blocking the task, even if you aren't blocking the thread. An actor that `thespis::Address::call`s another actor will not continue processing messages until that other actor sends back a response. Given that that actor might have a backlog of messages in it's queue, and it might itself wait for other actors whilst processing those, things could severely slow down. In practice this is not a problem if you have enough actors that can make progress at any given time. You will always have a maximal usage of the system's resources, but...

If you have a specific bottleneck, one or two actors that are crucial in your program that can not keep up with the rest, they become the slowest link in the chain.

Certain actors should spawn sub tasks rather than blocking on them. Eg. an actor that processes incoming requests from the network. If the back-end processing of those messages is async, this actor should probably continue to process other messages while the back-end generates a response. Again, it depends on the situation. If you count serving many connections concurrently and have one actor per connection, maybe it's fine for that actor to do but one thing at a time.

Note that `futures::SinkExt::send` also blocks the task, but only until the message has been delivered into the mailbox of the receiving actor. It doesn't wait until the message is processed, but will block if the mailbox is full.


## 5. Backpressure

In any concurrent system special care always must be taken when it comes to back pressure. If your system get's flooded with incoming messages faster than they can be processed, you can have unbounded memory consumption (unbounded channels), deadlocks (bounded channels, see above) as well as CPU exhaustion. You must have a conscious plan for how backpressure is provided to slow down incoming messages. Sometimes bounded channels can be a solution if you are careful not to deadlock. Other times you will need to build a custom back pressure mechanism to to limit the number of in flight requests (this is what _thespis_remote_ does).

You can also wrap the channel you pass to _thespis_impl_ with [_stream_throttle_](https://lib.rs/crates/stream_throttle).

[Handling overload](https://ferd.ca/handling-overload.html) is an interesting article from the Erlang world examining this rather difficult problem.


## 6. Actor Lifetime

Thespis follows a model very similar to the Rust memory model. The mailbox for an actor will stop running when all addresses to this actor have been dropped and all outstanding messages have been processed. This is neat, but it does oblige you to be very conscious about where addresses are lingering, or your program won't terminate, or you might have a memory leak.

The initial ambition was to say an address is always valid, and thus sending can conveniently be infallible. In practice this doesn't work, since an actor might panic while processing a message. When using remote actors, the network or the remote process might go down and on top of that, most channel implementations have a fallible send.

I kept this lifetime management, because usually having addresses around to actors that are no longer running is a sign of sloppy coding. _thespis_impl_ has support for a weak address, which will not keep the mailbox alive. This is particularly handy for situations like an actor which needs its own address, but shouldn't keep itself alive. Also be wary of two actors which have each other's address.

One way to sidestep this is to pull the plug. That is if you cancel the future running the mailbox, well it's terminated, however this is not recommended. If you really want an actor to stop it's own mailbox, regardless of other components that might still want to use it, you can let it spawn itself so it can have a `JoinHandle` that it can use to abort the mailbox.

[_futures::stream::abortable_](https://docs.rs/futures/0.3.15/futures/stream/fn.abortable.html) also let's you terminate a channel receiver manually. That's another way you can stop a mailbox.
