# What is the actor model?

The actor model is a program design model tailored to asynchronous concurrent processing. It was first conceived in 1973 by Carl Hewitt.  [Wikipedia](https://en.wikipedia.org/wiki/Actor_model) has a thorough introduction.

To simplify, the actor model is an object oriented model which encapsulates not only data but also behavior. That is an actor is an object which can hold local mutable state, but other actors cannot access the state, neither call methods on the actor. The only way to interact with it is by sending it a message.

This is the key advantage of the actor model. Since no direct method calls take place, there is no shared memory access. There is no need for locking as only the actor itself will be able to access it's state. The actor processes one message at a time. Thus during message processing it is always guaranteed to be the only piece of the program accessing it's state and no synchronization is needed.

You might have found out that people struggle to use the OOP model in Rust. Rust doesn't allow you to cheat when it comes to memory access. The compiler is very strict and it turns out its hard to write traditional OOP with a strict compiler. When you add concurrency and multi-threading to the mix it becomes downright impractical. The actor model solves exactly this problem as memory synchronization is simple eliminated from the equation.

For the rest of this book, it is assumed that you have some basic understanding of how the actor model works. If this is new for you, please check the following resources:

- [Wikipedia](https://en.wikipedia.org/wiki/Actor_model)
- [Everything you ever wanted to know about the actor model by Carl Hewitt](https://www.youtube.com/watch?v=7erJ1DV_Tlo) (video, highly recommended)


The Thespis API is inpired by [actix](https://docs.rs/actix/). If you are already familiar with actix, it will be easy to pick up thespis.
