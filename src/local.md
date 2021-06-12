# !Send Actors

In principle _thespis_ should be unconcerned by whether actors and message types are `Send`. Both types are user supplied and the user
has control over what executor they use and on what threads they run actors.

In practice it's not so simple. Rust requires one to be specific about `Send`ness in a number of type signatures, namely for a type in a `Box`. This means that we would have to double the whole interface and implementation if we were to support `!Send` messages. I have chosen not to do this for now.

A compromise is made with `!Send` actors. It turns out we can support them with reasonable boilerplate. For this, `Handler` has 2 methods, `handle` and `handle_local`. Mailbox implementations should provide a way to be spawned locally, and call `handle_local` in that case.

The interface keeps `handle` as a required method, but `handle_local` does call `handle` by default. This means there is no change in API for `Send` actors, but when you have a `!Send` actor, you must implement `handle_local` and provide an implementation for `handle` with an `unreachable`. This should never be called.
