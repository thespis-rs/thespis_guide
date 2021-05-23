# What is thespis?

When working on asynchronous projects in Rust I discovered the actor model thanks to [actix](https://docs.rs/actix/). I immediately found the model a very good fit for both Rust and asynchronous design. I started to implement software using the actix library and soon ran into a number of issues:

- it was in futures 0.1 (has been updated by now)
- doesn't work on wasm
- documentation was lackluster
- code base big and complicated
- didn't have remote actors
- my reasoning didn't seem to match with the actix lead developer

I tried briefly to evaluate what it would take to implement the features I wanted in actix, but it turned out to be complicated. Thus I set off rewriting from scratch. It has taken a long time, but I think the result is still relevant and worth it.

In the mean time, so many actor libraries have popped up that I have lost track of what each one exactly offers, you'll have to do your own research. Here is a list: https://lib.rs/search?q=actor

# Features

- __work on Wasm__: Note that async rust on embedded is still in it's infancy and I have not looked into it specifically, so no promises there for now. Thespis also uses boxing extensively, which might be a problem on `#[no_std]`.
- __minimal API design__ As an example compared to actix, there is no _MessageResponse_ trait, no _Context_ object, no _ActorFuture_, no DNS resolver, no _System_ nor _Arbiter_, no _ActorStream_, ...  The code base (without remote actors) is about 750 SLOC.
- __abstractions__: The address in thespis implements the _Sink_ trait, so you can easily chain it with streams. Make an actor observable with [_pharos_](https://crates.io/crates/pharos) and you have a pub-sub pattern. You can plug in the channel of your choice between the address and the mailbox.
- __full control__: you can interact at a low level with the address and the mailbox of an actor, you can change their implementation for your own as long as they implement the required traits.
- __remote actors__: _thespis_remote_ is a library that allows you to communicate with other processes and send messages to a remote process as if it was an actor in the local application. As far as I can tell, [_kay_](https://docs.rs/kay) is the only other Rust actor library that has remote actors.
- __supervision__: _thespis_impl_ supports supervision without even needing a `Supervisor` type.
- __executor agnostic__
- __well tested and documented__

# What's missing

- __message priority__: Since we work with `Sink` for the channel sender, which does not have an API for indicating the priority and the message is boxed inside the channel, it's not quite straightforward how this could be supported.
- __skipping messages__: The actor doesn't have access to the queue of messages. It can't observe them and decide what to process next.


In thespis, interface and implementation are 2 different libraries. The idea is that the interface defines the contract of what an actor is in terms of the Rust type system, but that you can swap out the implementation with something else yet remain compatible with other code that uses actors. You could for example write an implementation which uses actix under the hood if you wanted to create interop.
