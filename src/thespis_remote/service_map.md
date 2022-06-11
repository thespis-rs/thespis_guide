# ServiceMap

A service map is a type that knows how to deliver messages to individual actors. It dispatches if you will. You can tell `Peer` to expose a set of services remotely by bundling them in a `ServiceMap` and registering that map.

This makes it possible to expose different sets of services on different connections. For the moment it is not possible to dynamically change what services are exposed at runtime. The problem here is that we consider the service map a contract between two processes, so if all of a sudden we would stop providing certain services, the remote process might run into errors. This might change in the future, with some thought of how we inform a remote process of what services are available at any given time. One purpose might be to deal with authentication, enabling certain services only once a user has authenticated. However for now you have to deal with authentication yourself by adding a token in your messages. The receiving actor doesn't actually know which connection a message comes from.

There are three implementations of `ServiceMap` provided by _thespis_remote_:

- `service_map!`, a macro
- `RelayMap`, used for relaying messages to a different process.
- `PubSub`, for relays to implement a publish/subscribe architecture.


## `service_map!`

This is a macro because as it needs to deserialize your actor messages, it needs to know it's types, but that can only be provided inside your crate, not in _thespis_remote_. Thus it gives you all that functionality that needs to be in your crate to make things convenient so you don't have to write that yourself. It's main use looks like:

```rust
use
{
   thespis_remote :: { CborWF, service_map    } ,
   serde          :: { Serialize, Deserialize } ,
};

// Compared to local actor messages, remote once must also implement Serialize and Deserialize from Serde.
//
#[ derive( Serialize, Deserialize, Debug ) ] pub struct Add( pub i64 );
#[ derive( Serialize, Deserialize, Debug ) ] pub struct Show;

impl Message for Add  { type Return = ();  }
impl Message for Show { type Return = i64; }


// Remember that WireFormat is a trait. _thespis_remote_ is generic over the actual type, but it surely is part
// of the contract between 2 processes what wire format is being used. So you have to specify it for the macro.
//
service_map!
(
   namespace  : my_fancy_app_com ;
   wire_format: CborWF           ;
   services   : Add, Show        ;
);
```

The meaning of the parameters:
- **namespace**: this will be transformed into a module in your code. It is also used to uniquely identify services to avoid name collisions.
- **wire_format**: The wire format used for connections that expose this service map, in this case `CborWF`, the default implementation provided by _thespis_remote_.
- **services**: The message types that will be served by this service map.

**Usually you will put this macro as well as the message types mentioned in a separate crate that can be compiled into both processes that will communicate to eachother.**

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
#[ derive(Actor) ] pub struct Sum( pub i64 );

impl Handler<Add> for Sum
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
let addr_handler = Addr::builder().spawn( Sum(0), &AsyncStd )?;

// Create a service map.
//
let mut sm = my_fancy_app_com::Services::new();

// Register our handler. In this case the same actor will handle both types of messages.
//
sm.register_handler::<Add >( addr_handler.clone_box() );
sm.register_handler::<Show>( addr_handler.clone_box() );

// Register sm with a peer. See next chapter.
// ...
```

On the side that wants to use the service, you can obtain a `RemoteAddr` that accepts all the message types declared in the `service_map` macro. This cannot be statically verified by the compiler as the other side is generally another process. So you basically declare that you know that the process on the other end accepts messages of this type. Apart from this, everything is statically type checked in _thespis_.

```rust
// Call the service and receive the response.
//
let mut addr = my_fancy_app_com::RemoteAddr::new( peer_addr );

assert_eq!( Ok(()), addr.call( Add(5) ).await );
assert_eq!( Ok(()), addr.send( Add(5) ).await );
assert_eq!( Ok(10), addr.call( Show   ).await );
```

`RemoteAddr` works pretty much like `thespis::Addr`, except for the error type which will be `PeerErr` instead of `ThesErr` because a lot more things can go wrong when dealing with messaging over a network connection and you probably want to know what went wrong when it does.

# RelayMap

A `RelayMap` has the same place in the workflow as `Services` created by the `service_map!` macro, but instead of delivering to a local actor it passes on the message to a `Peer` that is connected to another process. This allows for transparent relaying to other backend services.

To create the `RelayMap`, you give it a `ServiceHandler` and a list of `ServiceID`s that should be relayed. Then you register it as a service map with the Peer that is listening for the incoming requests just as `Services`.

The `ServiceHandler` is an enum that is either an address to a `Peer` or a closure that will provide an address on a case by case basis. The latter option allows you to do load balancing or other runtime checks/logs before producing the address.

For a working code example, check the [relay example](https://github.com/thespis-rs/thespis_remote/tree/master/examples/relay) for _thespis_remote_.

# PubSub

- TODO
