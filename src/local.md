# !Send Actors

In principle _thespis_ should be unconcerned by whether actors and message types are `Send`. Both types are user supplied and the user
has control over what executor they use and on what threads they run actors.

In practice it's not so simple. Rust requires one to be specific about `Send`ness in a number of type signatures, namely for a type in a `Box`. This means that we would have to double the whole interface and implementation if we were to support `!Send` messages. I have chosen not to do this for now.

A compromise is made with `!Send` actors. It turns out we can support them with reasonable boilerplate. For this, `Handler` has 2 methods, `handle` and `handle_local`. Mailbox implementations should provide a way to be spawned locally, and call `handle_local` in that case.

The interface keeps `handle` as a required method, but `handle_local` does call `handle` by default. This means there is no change in API for `Send` actors, but when you have a `!Send` actor, you must implement `handle_local` and provide an implementation for `handle` with an `unreachable`. This should never be called.

## Example

This is a basic example of a `!Send` actor:

```rust
//! Spawn an actor that is !Send.
//
use
{
   thespis         :: { *                                         } ,
   thespis_impl    :: { Addr                                      } ,
   futures         :: { task::LocalSpawnExt, executor::LocalPool  } ,
   std             :: { marker::PhantomData, rc::Rc, error::Error } ,
};


#[ derive( Actor, Debug ) ] struct MyActor { i: u8, nosend: PhantomData<Rc<()>>}
#[ derive( Clone, Debug ) ] struct Ping( String )   ;


impl Message for Ping
{
   type Return = String;
}


impl MyActor
{
   async fn add( &mut self, mut x: u8 ) -> u8
   {
      x += 1;
      x
   }
}


impl Handler<Ping> for MyActor
{
   // Implement handle_local to enable !Send actor and mailbox.
   //
   #[async_fn_local] fn handle_local( &mut self, _msg: Ping ) -> String
   {
      // We can still access self across await points and mutably.
      //
      self.i = self.add( self.i ).await;
      dbg!( &self.i);
      "pong".into()
   }

   // It is necessary to provide handle in case of a !Send actor. It's a required method.
   // For Send actors that implement handle, handle_local is automatically provided.
   //
   #[async_fn] fn handle( &mut self, _msg: Ping ) -> String
   {
      unreachable!( "This actor is !Send and cannot be spawned on a threadpool" );
   }
}


fn main() -> Result< (), Box<dyn Error> >
{
   let mut pool = LocalPool::new();
   let     exec = pool.spawner();

   let actor    = MyActor { i: 3, nosend: PhantomData };
   let mut addr = Addr::builder().spawn_local( actor, &exec )?;

   exec.spawn_local( async move
   {
      let ping = Ping( "ping".into() );

      let result_local = addr.call( ping.clone() ).await.expect( "Call" );

      assert_eq!( "pong".to_string(), result_local );
      dbg!( result_local );

   })?;

   pool.run();

   Ok(())
}
```
