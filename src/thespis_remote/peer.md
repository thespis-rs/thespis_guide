# Peer

The `Peer` is the workhorse of _thespis_remote_. It manages a connection for you once you give it an `Stream + Sink` of the wireformat. It is generic over a `WireFormat`.

On the incoming path what it does looks like:

1. It spawns a loop that reads incoming messages from the connection and deserializes them into the wire format struct. If then looks at what type of message it is to apply back pressure to calls but no to other types of message.
2. This loop sends those to the `Peer` as messages, as the peer is an actor.
3. Unless the message is an error or a reponse to a request we made, we look up if we have a service map that handles this message type. We then spawn a task and tell the service map to deliver the message. This prevents the `Peer` from being blocked while a message is being processed. It means it can still process outgoing messages as well as process several incoming messages concurrently.

When an outgoing message is a call (request/response) the peer will send a oneshot channel back to the caller. Next it will store the sender of that channel with the `ConnID`, which later allows detecting that an incoming message is a response to this request and send the answer through the oneshot channel.

`Peer` does not do connection management like reconnect on loss etc. It is observable and will emit an event if the connection closes. It's up to you to handle that and potentially reconnect.

The easiest way to create the peer is from an `AsyncRead`/`AsyncWrite`. Wireformats must have a method for that. Lets have a look:

```rust
use 
{
	async_executors::AsyncStd,
	thespis_remote::{ CborWf },
};

let (peer, peer_mb, peer_addr) = CborWF::create_peer
( 
	"server"             , // a name for this connection
	tcp_connection       , // A transport that implements AsyncRead/AsyncWrite
	1024                 , // Max read message size for the codec (in bytes)
	1024                 , // Max write message size for the codec (in bytes)
	Arc::new( AsyncStd ) , // An executor to spawn tasks
	None                 , // an optional semaphore for backpressure
	None                 , // an optional grace period to finish outstanding tasks when closing down

).expect( "create peer" );
```

See the documentation on the WireFormat trait for some more explanation. You can also use the slightly more low level `Peer::new` if you want more control. That expects a `Stream`/`Sink` of the wire format as well as 2 thespis addresses. One for incoming messages and one for outgoing messages, as it allows you to set up a priority queue with the `futures::stream::select_with_strategy` combinator making sure outgoing messages are prioritized. This is especially useful if your process accepts requests to improve latency and lower memory consumption. It can also lower the risk of deadlocks, but that will be further explained in the chapter on backpressure.

Once the peer is created as shown above, we can continue the example from the chapter on `ServiceMap`.

```rust
use 
{
	async_executors::AsyncStd,
	std::sync::Arc,
};

// Register service map with peer
//
peer.register_services( Arc::new( sm ) );

// Start the peer actor
//
let peer_handle = AsyncStd.spawn_handle( peer_mb.start(peer) ).expect( "start mailbox of Peer" );

// Wait for the connection to close, which will automatically stop the peer.
//
peer_handle.await;

``` 
 

## Backpressure

`Peer` is a gateway in the sense described in the [earlier chapter on backpressure](https://thespis-rs.github.io/thespis_guide/thespis_impl/backpressure.html). It uses both a semaphore as well as a priority channel. 

Further more `Peer` spawns a new task for each incoming message. This guarantees it doesn't deadlock. As processing messages is asynchronous, and might even be relayed to other processes, this also means we can keep processing incoming messages concurrently rather than blocking and doing it serially.
