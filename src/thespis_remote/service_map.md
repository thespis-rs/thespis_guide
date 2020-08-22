# ServiceMap

A service map is a type that knows how to deliver messages to individual actors. It dispatches if you will. You can tell `Peer` to expose a set of services remotely by bundling them in a `ServiceMap` and registering that map.

This makes it possible to expose different sets of services on different connections. For the moment it is not possible to dynamically change what services are exposed at runtime. The problem here is that we consider the service map a contract between two processes, so if all of a sudden we would stop providing certain services, the remote process might run into errors. This might change in the future, with some thought of how we inform a remote process of what services are available at any given time. One purpose might be to deal with authentication, enabling certain services only once a user has authenticated.

There are two implementations of `ServiceMap` provided with _thespis_remote_:

- `service_map!`, a macro
- `RelayMap`, used for relaying messages to a different process.


## `service_map!`

This is a macro because as it needs to deserialize your actor messages, it needs to know it's types, but that can only be provided inside your crate, not in _thespis_remote_. It's main use looks like:

```rust
use
{
   thespis_remote :: { ThesWF, service_map    } ,
   serde          :: { Serialize, Deserialize } ,
};

// Compared to local actor messages, remote once must also implement Serialize and Deserialize.
//
#[ derive( Serialize, Deserialize, Debug ) ] pub struct Add( pub i64 );
#[ derive( Serialize, Deserialize, Debug ) ] pub struct Show;

impl Message for Add  { type Return = ();  }
impl Message for Show { type Return = i64; }


service_map!
(
   namespace  : my_fancy_app_com ;
   wire_format: ThesWF           ;
   services   : Add, Show        ;
);
```

The meaning of the parameters:
- namespace: this will be transformed into a module in your code. It is useful as well to uniquely distinguish services that might have name collisions otherwise.
- wire_format: The wire format used for connections that expose this service map, in this case `ThesWF`, the default implementation provided by _thespis_remote_.
- services: The message types that will be served by this service map.

Usually you will put this macro as well as the message types mentioned in a separate crate that can be compiled into both processes that will communicate to eachother.

You can of course interact with such process also from binaries written in different languages as long as they correctly speak the wire format.

Usage of `service_map!` looks like this on the side that exposes these services:


```rust
use
{
   thespis         :: { Actor, async_fn } ,
   thespis_impl    :: { Addr            } ,
   async_executors :: { AsyncStd        } ,
};

// Let's imagine a simple actor that can receive `Sum` and `Show`.
//
#[ derive( Actor ) ] pub struct Sum( pub i64 );

impl Handler< Add > for Sum
{
   #[async_fn] fn handle( &mut self, msg: Add ) -> ()
   {
      self.0 += msg.0;
   }
}


impl Handler< Show > for Sum
{
   #[async_fn] fn handle( &mut self, _msg: Show ) -> i64
   {
      self.0
   }
}

// Create mailbox for our handler and start it using async-std as the executor.
// The type of addr_handler is `Addr<Sum>`.
//
let addr_handler = Addr::builder().start( Sum(0), &AsyncStd ).expect( "spawn actor mailbox" );

// Create a service map.
//
let mut sm = my_fancy_app_com::Services::new();

// Register our handler.
//
sm.register_handler::<Add >( addr_handler.clone_box() );
sm.register_handler::<Show>( addr_handler.clone_box() );

// Register sm with a peer.
// ...
```

On the side that wants to use the service, after creating the peer, it's completely transparent. The `RemoteAdd` will accept all the message types declared in the `service_map` macro.

```rust
// Call the service and receive the response.
//
let mut addr = my_fancy_app_com::RemoteAddr::new( peer_addr );

assert_eq!( Ok(()), addr.call( Add(5) ).await );
assert_eq!( Ok(()), addr.send( Add(5) ).await );
assert_eq!( Ok(10), addr.call( Show   ).await );
```
