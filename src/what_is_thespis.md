# What is thespis?

When working on asynchronous projects in Rust I discovered the actor model thanks to [actix](https://docs.rs/actix/). I immediately found the model a very good fit for both Rust and asynchronous design. I started to implement software using the actix library and soon ran into a number of issues:

- it was in futures 0.1 (has been updated by now)
- doesn't work on wasm
- documentation was lackluster
- code base big and complicated
- didn't have remote actors
- my reasoning didn't seem to match with the actix lead developer

I tried briefly to evaluate what it would take to implement the features I wanted in actix, but it turned out to be complicated. Thus I set off rewriting from scratch. It has taken a long time, but I think the result is still relevant and worth it.

In the mean time, so many actor libraries have popped up that I have lost track of what each one exactly offers, you'll have to do your own research. Here is a list: [lib.rs/search?q=actor](https://lib.rs/search?q=actor).

# Features

- __Interfaces__: Interface and implementation are separate. The behavior of an actor is described in the _thespis_ crate with traits. _thespis_impl_ has a reference implementation. However, libraries can expose an actor API without having to pull in an implementation, just by implementing the traits. Clients can choose their implementation and aren't obliged to use the reference implementation.
- __works on Wasm__: Note that async rust on embedded is still in it's infancy and I have not looked into it specifically, so no promises there for now. Thespis also uses boxing extensively, which might be a problem on `#[no_std]`.
- __minimal API design__ As an example compared to actix, there is no `MessageResponse` trait, no `Context` object, no `ActorFuture`, no DNS resolver, no `System` nor `Arbiter`, no `ActorStream`, ... yet it does about everything you can do with actix and more. The code base (without remote actors) is about 1.2k SLOC.
- __abstractions__: The address in thespis implements the _Sink_ trait, so you can easily chain it with streams. Make an actor observable with [_pharos_](https://crates.io/crates/pharos) and you have a pub-sub pattern. You can plug in the channel of your choice between the address and the `Mailbox` (`Mailbox` only requires you give it a `Stream`). You can even have different channels for different addresses to the same `Mailbox`. You can use combinators on the receiving end to have different priorities depending on the address.
- __full control__: you can interact at a low level with the address and the mailbox of an actor, you can change their implementation for your own as long as they implement the required traits.
- __remote actors__: _thespis_remote_ is a library that allows you to communicate with other processes and send messages to a remote process as if it was an actor in the local application. As far as I can tell, [_kay_](https://docs.rs/kay) is the only other Rust actor library that has remote actors.
- __supervision__: _thespis_impl_ supports supervision without even needing a `Supervisor` type.
- __performant__: I need to run some more tests, but we seem to be about twice as fast as actix in most scenarios. There is one notable exception. Under contention, that is one receiver with many senders on different threads, actix is about 3 times faster. Somehow they spread more work on the sender threads, and in this case the receiver thread is the bottleneck, so that works really well.
- __executor agnostic__
- __well tested and documented__

# What's missing

- __skipping messages__: The actor doesn't have access to the queue of messages. It can't observe them and decide what to process next. You could of course implement this by having a channel that allows for inspection of what's in it's buffer, but the `Mailbox` of `thespis_impl` does not deal with this for you.

In thespis, interface and implementation are 2 different libraries. The idea is that the interface defines the contract of what an actor is in terms of the Rust type system, but that you can swap out the implementation with something else yet remain compatible with other code that uses actors. You could for example write an implementation which uses actix under the hood if you wanted to create interop.
