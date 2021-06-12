# Abstract the Actor type

The last chapters cover the most straight forward use of thespis. But how does it all compose? What if need to store a list of addresses to different actors that accept a certain type of message? What if I have to send 2 different message types to this heterogeneous list? `Addr` is actually `Addr<A>`. It's generic over a type of actor, thus it can send all the message types this actor implements `Handler` for, but that does mean we can not store a list of addresses to different actors.

The solution to this is the trait `Address<M>` from the interface library _thespis_. We can store a `Box< dyn Address<M, Error=E> + Send + Unpin >` and thus we can store addresses to different actors in the same collection. Because this type is unwieldy, thespis has a shorthand: `BoxAddress<M, E>`. The error type comes from the channel Sender. _thespis_impl_ wraps the errors from different channels in `ThesErr`, so generally if you use the `Addr` type, that's the error type you should use.

Of course, if your actor types are known at compile time and you don't have to many of them you can also use an enum:

```rust
enum Postman
{
   Foo( Addr<Foo> ),
   Bar( Addr<Bar> ),
}
```
However, you will have to match and de-structure the enum to actually send anything. And mind you that sometimes boxing doesn't have any measurable overhead, so the inconvenience might well not be worth it.

# Multi dimensional

But what if you have several message types and you will have to send them to several types of actors? Unfortunately we [cannot currently express](https://github.com/rust-lang/rfcs/issues/2035) `Box< dyn Address<Foo> + Address<Bar> >` in rust. There is a piece of boilerplate that solves this however:

```rust
struct Foo;
struct Bar;

impl Message for Foo { type Return = (); }
impl Message for Bar { type Return = (); }


trait AddressFooBar: Address<Foo, Error=ThesErr> + Address<Bar, Error=ThesErr> {}

impl<T> AddressFooBar for T

   where Self: Address<Foo, Error=ThesErr> + Address<Bar, Error=ThesErr>
{}
```

Now we can have `Box< dyn AddressFooBar >` and be able to send both `Foo` and `Bar` messages. The [trait_set](https::crates.io/crates/trait_set) crate can make this more streamlined.

# Unknown unknowns

Sometimes it's not even known in your library which types of actors and which types of messages you will have to handle, because they are user defined. Yet you still need to store a bunch addresses. You will need some other information at runtime that indicates which type the actor and message are, to enable downcasting.

In this case you can store the addresses as `Box< dyn Any >` and then downcast them at runtime to the right type. We can store a collection eg. `HashMap` of `Box< dyn Any >` and downcast them to `Box< dyn Address<M, E> >`. You will need to store (eg. in the keys of your hashmap) what type you are actually holding to downcast it. See the [recipient_any example](https://github.com/thespis-rs/thespis_remote/tree/master/examples/recipient_any.rs).


# Cloning trait objects

The clone trait is not object safe in Rust, so you can't make a trait object from a trait that requires implementers to also implement `Clone`. For this reason, the `Address` trait has a `clone_box` method that requires implementors to be able to produce a `BoxAddress<M>`. This way if you have a boxed address you can still clone it conveniently.
