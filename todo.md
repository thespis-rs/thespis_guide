## Guide level docs

- think about and write up the difference in Send + reply_to and Call. Performance, possiblity to pass on reply_to, link the outgoing send to the incoming response? Possibility to send a reply_to address that is not your own?


### Design patterns
- how can you stop a mailbox before all adresses are dropped? -> drop the future or the joinhandle
- how can an actor stop it's own mailbox -> panic/spawn itself, so it can have the joinhandle/send it it's joinhandle through a message.
- how can an actor send messages to itself? -> give it it's address in it's constructor. Be careful that it keeps itself alive and must get an outside signal to drop this address, or you drop the mailbox.
- concurrent message processing, make an actor without mutable state and clone it to run it in parallel with a load balancer in front, or just spawn tasks in your handler that do the actual work so you can immediately take the next message without blocking the actor.
- concurrent message processing with update self (without actorfuture) -> pass actor address to spawned subtask so it can send a message back for updating state. Potentially you may want to make the subtask an actor itself.
- interfaces: take an address to an actor that implements Handler<X> + Handler<Y> + ...?
