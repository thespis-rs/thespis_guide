# Backpressure

Backpressure is an important concept in asynchronous message based systems. Backpressure allows receivers to slow down senders when they cannot process messages/requests fast enough. This limits the memory consumption and improves latency.

The most basic form of back pressure in thespis can be achieved by using bounded channels for mailboxes. This will make sender tasks wait before putting messages in a mailbox that is full. Thereby slowing down parts of the process that are outpacing other parts. There is a number of situations in which this can work well, but there is one particular trap. By creating a circular shape with one actor acting as a gate to the outside world, receiving and sending to the outside, you can create a deadlock.

The issue is described at length with illustrations in [this article](https://elizarov.medium.com/deadlocks-in-non-hierarchical-csp-e5910d137cc). Please go read that and then come back.

The solution to this problem in it's simplest form is indeed to have a separate outgoing queue for the gateway actor. In thespis this is relatively simple with the use of `futures::stream::select_with_strategy`. There is also a demonstration of this in the [examples directory](https://github.com/thespis-rs/thespis_impl/blob/dev/examples/deadlock_prio.rs).

However, this is only the most simple form of this problem. This is a closed circuit with one node that acts as a gateway. Eg. manages the incoming and outgoing messages. So when we have a priority queue, the incoming messages do not fill up the outgoing queue and as soon as the processing actor (or pipeline for that matter) can offload a response, they will get a new message from their mailbox. This free slot will propagate all the way back to the gateway. The gateway can now offload the current message to the processing actor and will now processe the outgoing message first (as it'a priority queue). Thus, the system cannot deadlock.

However imagine the gateway is a client connection to our server. And imagine we can have more than 1 client connected at the same time (well, it's a big motivator for async architecture, isn't it). Now our solution no longer works. As now all the client connections are competing for new free slots in the processing actor/pipeline. If it so happens that the gateway which received outgoing messages in their outgoing queue loses this competition, it's possible that their outgoing queue fills up and hopla. Deadlock 2.0.

A solution for this problem is to forgo the mailboxes for back pressure and to use a separate system, like a semaphore. Gateways need permits before sending more requests into the system. This works, but it's not trivial to define exactly what is the right number of permits here, especially if someone might come along later and change the architecture of the system.

Also, it only solves our exact problem. Yes, things can get worse. What if the processing pipeline isn't exactly linear, but a complex application? What if some of those actors can actually create new messages that end up in the system? You'd have to make absolutely sure that they reserve a slot in the semaphore before doing so, if not the semaphore no longer accurately keeps track of the number of messages that are currently in the system. What if they can create new actors? 

An oft suggested solution to all of this is to use unbounded channels. There are however several problems with them. First of all, it means no more backpressure. In itself that's fine as we established that using the mailbox as a backpressure mechanism in this case is a bad idea. However more fundamentally the problem is that unbounded channels don't exist. They would require unbounded memory, which most systems don't have, as well as create unbounded latency which is generally not desirable. 

Another solution is to let the gateway spawn a new task for every incoming message, making it completely resistant against this type of deadlock. However here an external back pressure mechanism like a semaphore becomes really important, otherwise we have a situation similar to unbounded channels, which doesn't really work.

There is in my opinion no one size fits all solution here. You must carefully think your architecture through, and be conscious about your queue-sizes.

