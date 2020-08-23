# Desugar Addr::builder()

Even if you will mostly be using the convenience builder to start your actors, its good to have an understanding of how things work at a lower level. So what would things look like if we wanted to manually create an actor? We will keep the simple example from the last chapter but desugar it. We already showed the desugaring of the `async_fn` macro. Here we show a full working example with the main function as desugared as we can. Just know that there are intermediate steps on the builder API if you only want to override certain parameters.

```rust
use
{
   thespis         :: { *                         } ,
   thespis_impl    :: { *                         } ,
   async_executors :: { AsyncStd, SpawnHandleExt  } ,
   std             :: { error::Error              } ,
   futures         :: { channel::mpsc             } ,
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
   let (tx, rx)  = mpsc::channel( 5 );
   let tx        = Box::new( tx.sink_map_err( |e| Box::new(e) as SinkError ) );
   let mb        = Mailbox::new( Some("HelloWorld".into()), Box::new(rx) );
   let mut addr  = Addr::new( mb.id(), mb.name(), tx );
   let actor     = MyActor;

   let handle = AsyncStd.spawn_handle( mb.start( actor ) )?;

   let result = addr.call( Hello( "hello".into() ) ).await?;

   assert_eq!( "world", dbg!(result) );

   drop( addr );
   handle.await;

   Ok(())
}
```

## Channels

In thespis you can choose what channel will be used for communication between the address and the mailbox. Out of the box the builder supports both tokio and futures channels, in bounded and unbounded forms. But you might want to try a different channel type. One interesting application is if you want to use a channel that overwrites older messages instead of providing back pressure, like the one in the _ring-channel_ crate.

The abstractions the thespis types work on are defined as:
```rust

/// Interface for T: Sink + Clone
//
pub trait CloneSink<'a, Item, E>: Sink<Item, Error=E> + Unpin + Send
{
   /// Clone this sink.
   //
   fn clone_sink( &self ) -> Box< dyn CloneSink<'a, Item, E> + 'a >;
}


impl<'a, T, Item, E> CloneSink<'a, Item, E> for T

   where T: 'a + Sink<Item, Error=E> + Clone + Unpin + Send + ?Sized

{
   fn clone_sink( &self ) -> Box< dyn CloneSink<'a, Item, E> + 'a >
   {
      Box::new( self.clone() )
   }
}


/// A boxed error type for the sink
//
pub type SinkError = Box< dyn std::error::Error + Send + 'static >;

/// Type of boxed channel sender for Addr.
//
pub type ChanSender<A> = Box< dyn CloneSink< 'static, BoxEnvelope<A>, SinkError> >;

/// Type of boxed channel receiver for Mailbox.
//
pub type ChanReceiver<A> = Box< dyn futures::Stream<Item=BoxEnvelope<A>> + Send + Unpin >;
```

Thus the sender must implement `Sink`, `Clone`, `Unpin` and `Send` and it's error type must be `Box< dyn std::error::Error + Send + 'static >`. The receiver must be infallible and implement `Stream`, `Send` and `Unpin`. If your favorite channel implementation does not provide the `Sink` interface on it's sender, check out the `async_chanx` crate which provides some `Sink` implementations for you.

## Mailbox and Addr

Next we create our mailbox and address:

```rust
let mb       = Mailbox::new( Some("HelloWorld".into()), Box::new(rx) );
let mut addr = Addr::new( mb.id(), mb.name(), tx );
```

You can see that the constructors take a few parameters. The first parameter on `Mailbox::new` is an optional name. This will be used in logging to help you identify which actor is doing what. The id in the second line is similar but is just a numeric counter. Every mailbox created in the process will have a unique numeric id, but you can also name them to make it easier to understand your logs. `Addr` also exposes both the `id` and `name`. If two addresses return the same `id`, they both talk to the same mailbox and thus actor.

We feed both ends of the channel to the constructors of `Mailbox` and `Addr`. You may notice that we just created both the mailbox and the address without even having instantiated an actor yet. That is because we only really need an actor when we start the mailbox. This has an interesting property. If we want our actor to have a copy of it's own address, we can just pass it along in it's constructor, since as soon as we start it we can only communicate to it through messages. It would be a bit of a pain to have to create a specific message type to give the actor it's own address.

## Starting the mailbox

```rust
let actor  = MyActor;
let handle = AsyncStd.spawn_handle( mb.start( actor ) )?;
```

Next we create an actor and pass it to `Mailbox::start`. This method returns a future that you can spawn however way you like on the executor of your choice. `Mailbox` has a a few convenience methods as well which allow you to pass the executor by reference which avoids having to import the spawn traits as well as local versions for spawning actors that aren't `Send`. Check the API docs.

We use the `AsyncStd` wrapper from _async_executors_ here. One essential difference with using `async-std` directly is that the joinhandle this returns will cancel the future when you drop it, which is a way you can stop an actor. The recommended way is to drop all addresses to the actor.

## Stopping the actor

```rust
drop( addr );
handle.await;
```

As described above, to drop an actor we drop all addresses to it. Now we can await the mailbox which will terminate.
