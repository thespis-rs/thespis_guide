# Debugging

Debugging asynchronous programs can be a bit of a challenge. _thespis_impl_ tries to give you the relevant information and tooling to succeed in seeing exactly what is going on in your program. The main feature is generating useful logs about what's going on so you can check what each concurrent component is doing.

_thespis_impl_ uses [_tracing_](https://crates.io/crates/tracing) for it's logging. Its possibility of instrumenting futures and executors with spans (through [_tracing-subscriber_](https://crates.io/crates/tracing-subscriber)) makes it uniquely suitable for logging in async applications.

Whenever the `Mailbox` runs, it will enter a spawn identifying your actor by id and name. Log events within `Addr` and `WeakAddr` will also be within such span. This will help you identify which actor messages are coming from. It is important you choose a subscriber that prints span information.

In order to turn on logging, setup a basic tracing subscriber. You can add this to the beginning of your main function:

```rust
let _ = tracing_subscriber::fmt::Subscriber::builder().init();
```

One thing to note is that if within your handler you spawn on an executor, you will lose the association with the current spawn. However you can use `tracing::current_span()` to get the span and now you can use `tracing-futures` to instrument the future you want to spawn. Alternatively you can create a new span and set the actor's span as the parent.

In any case, anything you log in your handler will be tagged with the actor's span, but if you spawn new tasks, you will have to manually include a span if you want to avoid losing the association with the actor.


## Using the log crate

If your application uses the log crate, but you want to receive the log events emitted by _thespis_impl_, please refer to [the relevant documentation of the tracing crate](https://docs.rs/tracing/0.1.26/tracing/index.html#log-compatibility).


## Visualizing logs

The usual problem with async logs is that you will get all log lines interleaved. This makes it very hard to reason about what is going on. The tracing spans _thespis_impl_ adds allow us to know which actor a log event is associated with. This means we can separate out logs per actor and put them side by side.

I wrote a little web app called [tracing_prism](https://github.com/najamelan/tracing_prism) that visualizes your logs in columns side by side. For best results, generate your logs in json format:

```rust
let _ = tracing_subscriber::fmt::Subscriber::builder()
	.json()
	.init()
;
```


