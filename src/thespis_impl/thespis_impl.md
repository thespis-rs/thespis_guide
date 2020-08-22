# thespis_impl

This is the reference implementation. This chapter goes over it's features and provides you with example code. The most basic example, hello world:

```rust
use
{
	thespis         :: { *            } ,
	thespis_impl    :: { *            } ,
	async_executors :: { AsyncStd     } ,
	std             :: { error::Error } ,
};


#[ derive( Actor ) ]
//
struct MyActor;


struct Hello( String );

impl Message for Hello
{
	type Return = String;
}


impl Handler< Hello > for MyActor
{
	#[async_fn]	fn handle( &mut self, _msg: Hello ) -> String
	{
		"world".into()
	}
}


#[async_std::main]
//
async fn main() -> Result< (), Box<dyn Error> >
{
	let mut addr = Addr::builder().start( MyActor, &AsyncStd )?;

	let result = addr.call( Hello( "hello".into() ) ).await?;

	assert_eq!( "world", result );

	Ok(())
}
```

## Discussion

Let's quickly take a tour of the anatomy of this simple program:

```rust
#[ derive( Actor ) ]
//
struct MyActor;
```

`Actor` is a trait defined in the _thespis_ crate. It has no required methods, so you can easily derive it. `MyActor` here is what generally holds the (mutable) state of your actor. In this simple example there is no state. The mailbox will take ownership of this and after that you can only communicate with it by sending messages through the address you get back. Once you give it to the mailbox you can no longer call methods on it.

```rust
struct Hello( String );

impl Message for Hello
{
	type Return = String;
}
```

`Hello` is a message type. The type system will guarantee that you can never send a message type to an actor unless it implements `Handler` for that type and the type implements the `Message` trait. As you will have to implement this trait for your message types, you will have to wrap types that are not defined in your crate in order to use them as a message. Here we wrap `String`.

The associated type is the return type of the handler. When using `Addr::call` your actor can return a value to the caller, making it easy to implement request-response type communication, mimicking a method call. Sending a message to an actor is always asynchronous.

```rust
impl Handler< Hello > for MyActor
{
	#[async_fn]	fn handle( &mut self, _msg: Hello ) -> String
	{
		"world".into()
	}
}
```

Here we define that `MyActor` can process messages of type `Hello`. The body of the function does the actual processing. As you can see it receives a `&mut self`, even though we know that all messages are sent asynchronously. This is the main advantage of the actor model. Even though any place in your code that has this actor's address can easily send messages, you never need thread sync like locks on your data. Only this actor can access it's own state directly and the only way to communicate with it is through sending messages. Further more the mailbox will make the actor process one message at a time, so there is never shared mutual access the the state.

Using plain Object Oriented Programming in async Rust with methods that access state is very difficult, since as soon as you spawn any task, that task cannot hold any references to anything outside of it and to make matters worse, you can never hold a mutex across an await point.

The `async_fn` macro deals with the fact that Rust doesn't support async trait methods at the moment. It does this in a very similar way as the _async-trait_ crate, but it outputs much simpler code, making it compatible with a hand written version of this method, which was not possible with _async-trait_.

A handwritten version would look like:

```rust
impl Handler< Hello > for MyActor
{
	fn handle( &mut self, _msg: Hello ) -> Return<'_, String>
	{
		Box::pin( async
		{
			"world".into()
		})
	}
}
```

Where `Return` is defined as:

```rust
/// A boxed future that is `Send`, shorthand for async trait method return types.
//
pub type Return<'a, R> = Pin<Box< dyn Future<Output = R> + 'a + Send >>;
```

Note that within this handler you can `await` as it is asynchronous, but that while it is waiting, this actor will not process any more messages.

```rust
let mut addr = Addr::builder().start( MyActor, &AsyncStd )?;
```

We use a convenience builder to create the address we will use to send messages to our actor, and tell it to
generate a default mailbox for it and spawn it on the provided executor.

The `AsyncStd` type comes from the _async_executor_ crate which provides a uniform interface for executors,
allowing us to be executor agnostic. We could just as well have given it a `tokio` executor or one from the `futures` crate.

Note that this function takes our actor by value as we shouldn't access it anymore directly once it starts processing messages.

```rust
let result = addr.call( Ping( "hello".into() ) ).await?;
```

We use `Address::call` to send a message to our actor. This method will return to us a future that will resolve to the answer our handler returns. Note that `Addr` also implements `futures_sink::Sink`. The `send` method from the `Sink` trait will drop the returned value and will return to us as soon as the message is delivered to the mailbox, without waiting for the actor to process the message.

Thus you can also use the `call` method even if you don't want to return any value to be sure that the message has been processed, where as `send` is more like throwing a message in a bottle. You will still get back pressure from `send` as it will block when the channel between the `Addr` and the mailbox is full.

In the next chapter we will take a look at desugaring the builder and manually create our mailbox and address.
