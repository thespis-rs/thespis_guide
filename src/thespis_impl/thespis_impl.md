# thespis_impl

This is the reference implementation for the interface defined in the _thespis_ crate. This chapter goes over it's features and provides you with example code. The most basic example, hello world:

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
   #[async_fn] fn handle( &mut self, _msg: Hello ) -> String
   {
      "world".into()
   }
}


#[async_std::main]
//
async fn main() -> Result< (), Box<dyn Error> >
{
   // .start here spawns your mailbox/actor immediately on the given executor and
   // detaches the joinhandle. You can also use the `spawn..` functions on the builder
   // in order to get a JoinHandle which you should await as it will drop the mailbox
   // when dropped.
   //
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

`Actor` is a trait defined in the _thespis_ crate. It has no required methods, so you can easily derive it. `MyActor` here is what generally holds the (mutable) state of your actor. In this simple example there is no state, but otherwise you could manipulated it from within the implementation of `Handler<T>`. The mailbox will take ownership of your actor and after that you can only communicate with it by sending messages through the address you get back. Once you give it to the mailbox you can no longer call methods on it.

```rust
struct Hello( String );

impl Message for Hello
{
   type Return = String;
}
```

`Hello` is a message type. The type system will guarantee that you can never send a message type to an actor unless it implements `Handler` for that type and the type implements the `Message` trait. As you will have to implement this trait for your message types, you will have to wrap types that are not defined in your crate in order to use them as a message. Here we wrap `String`. If your handler might panic, please make sure the message type is `UnwindSafe`, as the mailbox will call `catch_unwind` on the handler method. This allows us to elegantly allow supervising of actors. All together it is recommended that your handlers don't panic, rather return a `Result` if they need to be fallible. Nevertheless, messages in the actor model are meant to be data and not have any shared resources like locks or references in them.

The associated type is the return type of the handler. When using `Addr::call` your actor can return a value to the caller, making it easy to implement request-response type communication, mimicking a method call. Sending a message to an actor is always asynchronous. Note: we could also have written `-> <Hello as Message>::Return` as the return type here. In any case, it needs to be the same type.

```rust
impl Handler< Hello > for MyActor
{
   #[async_fn] fn handle( &mut self, _msg: Hello ) -> String
   {
      "world".into()
   }
}
```

Here we define that `MyActor` can process messages of type `Hello`. The body of the function does the actual processing. As you can see it receives a `&mut self`, even though we know that all messages are sent asynchronously. This is the main advantage of the actor model. Even though any place in your code that has this actor's address can easily send messages, you never need thread sync like locks or the infamous `Rc<Refcell>` on your data. Only this actor can access it's own state directly and the only way to communicate with it is through sending messages. Further more the mailbox will make the actor process one message at a time, so there is never shared access to the mutual state.

Using plain Object Oriented Programming in async Rust with methods that access state is very difficult, since as soon as you spawn any task, that task cannot hold any references to anything outside of it and to make matters worse, you shouldn't hold a mutex across an await point. The actor model sidesteps these problems, as any code that needs to communicate with an actor only needs the address, not a reference to the actor itself. `Addr` implements clone, so you can have many places of your program talk to the actor.

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
allowing us to be executor agnostic. We could just as well have given it a _tokio_ executor or one from the _futures_ crate, or _wasm-bindgen_ on Wasm.

Note that this function takes our actor by value as we shouldn't access it anymore directly once it starts processing messages.

```rust
let result = addr.call( Ping( "hello".into() ) ).await?;
```

We use `Address::call` to send a message to our actor. This method will return to us a future that will resolve to the answer our handler returns. Note that `Addr` also implements `futures_sink::Sink`. You can use the combinators from the futures crate to forward an entire stream into the address, as long as the actor implements `Handler` for the type the stream produces. The `send` method from the `Sink` trait will drop the returned value and will return to us as soon as the message is delivered to the mailbox, without waiting for the actor to process the message.

Thus you can also use the `call` method even if you don't want to return any value to be sure that the message has been processed, where as `send` is more like throwing a message in a bottle. You will still get back pressure from `send` as it will block when the channel between the `Addr` and the mailbox is full (as long as it's not an unbounded channel that is).

In the next chapter we will take a look at desugaring the builder and manually create our mailbox and address.


## Weak and Strong addresses.

It doesn't figure in the basic example, but the mailbox of the actor stops when all addresses to it are dropped. The `JoinHandle` from the executor will return your actor to you if you want to re-use it later. When the mailbox is stopped because your actor panicked, you will retrieve the mailbox instead and you can instantiate a new actor and spawn it on the same mailbox, so all addresses remain valid. This is further elaborated in the chapter about supervision.

_Thespis_impl_ also has weak addresses. These addresses don't keep the mailbox alive. It is handy when for example the actor needs it's own address. In this case you don't necessarily want it to keep itself alive. Creating a weak address is simple:

```rust
// Does not consume addr, hence it's not called downgrade.
//
let weak_addr = addr.weak();

// You can create a strong address from a weak one as long as the mailbox is still open.
// If all strong addresses were already dropped at this point, you will get an error instead.
//
let strong = match weak.strong()
{
   Ok(addr) => addr,
   Err(e)   => {} // -> ThesErr::MailboxClosed.
};
```


