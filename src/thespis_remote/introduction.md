# Introduction to thespis_remote.

_Thespis_remote_ adds support for remote actors to the thespis actor model. It follows the same rather low level approach as the rest of _thespis_. That means it provides the ability to send actor messages over one connection. The connection _thespis_remote_ works over is a `Sink` and `Stream` of a given wire format. The wire format has an encoder/decoder so works over anything that implements `AsyncRead`/`AsyncWrite`.

There is no "system" or service discovery integrated. It requires you to connect two processes by any means of your choice and provide the above interfaces to _thespis_remote_. From there _thespis_remote_ provides:

- `RemoteAddr` which allows you to send messages as if you were sending to an actor in the same process (only difference, `send` does not guarantee in order delivery).
- `ServiceMap`, a trait and provided reference impl that knows how to deserialize your actor messages and dispatch them to the right actor. Note that you can only set one actor per message type per service map, so you are actually sending to a remote process rather than to an individual actor in that process. However you can easily wrap your message with an actor id and provide more precise delivery if you want.
- `WireFormat`, a trait and reference impl for the actual wire format used.
- `Peer`, an actor that manages a connection endpoint. You provide it with a service map to "serve" over the connection and for sending messages, you pass it's address to the constructor of `RemoteAddr`, implying that you know that the other side of the connection managed by this `Peer` exposes these services. This last part is the only thing we cannot guarantee in the type system, as the compiler does not know about the process on the other end of your connection.

These are the four core components of the library.

_thespis_remote_ also provides relaying of messages. Further some features are included to deal with back pressure and to make it resistant to attacks like DOS and slow loris.

Apart from this guide you can also consult the [API documentation](https://docs.rs/thespis_remote) and the [examples](https://github.com/thespis-rs/thespis_remote/tree/master/examples).


# Scope

The scope of _thespis_remote_ is specifically as described above. On the low end of the scope this means there is no management of your connection. When the connection fails, `Peer` will generate an event for you to react on. You have to re-connect and re-setup your `Peer` and service map. This limitation is necessary to keep the scope of the project reasonable as many different types of connections can be used. You can obviously write your own wrapper that provides automatic reconnect and maybe in the future something like that will be provided by the thespis project, but it won't be in _thespis_remote_.

On the high end of the scope, _thespis_remote_ does not handle any application logic. Most prominently authentication/authorization comes to mind. What if a user has to authenticate before being allowed to send certain messages to your service. It generally comes down to 2 key components: prevent MITM by end to end encryption (do this on the connection level) and sending along some proof of identity (think cookie in HTTP land). _thespis_remote_ does not have any instrumentation for this and it will be up to you to handle this in your message types.

You might feel cheated that you can't address individual actors on a remote process. There are good reasons for this however and nothing stops you from implementing individual actor addressing on top of _thespis_remote_. In _thespis_ everything is as much as possible statically type checked. By considering a process that exposes it's possibility to receive messages of certain types, we adhere to a public interface like a contract. It allows a remote process to statically create addresses to send us messages, rather than having to send over addresses at runtime.

Actors can accept several types of messages, but some of those might not be meant for external use. If we were to expose an address to a specific actor type as is, all of it's interface would be remotely exposed. We would also need to compile in the actor type in the remote process, where with the current design only message types need to be shared between the two processes.

_thespis_remote_ is minimal in the sense that it creates the fundamental building block for remote actor communication. Systems, discovery and individual actor addressing can all be implemented on top of it.

Several components of the library can easily be customized. The traits `WireFormat` and `ServiceMap` allow you to create your own specialized implementations if desired.


