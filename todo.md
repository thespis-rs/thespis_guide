## Guide level docs

- chapters on error handling, both for thespis_impl and thespis_remote, explaining a high level overview of the error handling, like what to use pharos for and how errors are logged and sent to the remote.
- think about and write up the difference in Send + reply_to and Call. Performance, possiblity to pass on reply_to, link the outgoing send to the incoming response? Possibility to send a reply_to address that is not your own?


### Design patterns
- how can an actor stop it's own mailbox -> panic/spawn itself, so it can have the joinhandle/send it it's joinhandle through a message.
- how can an actor send messages to itself? -> give it it's address in it's constructor. Be careful that it keeps itself alive and must get an outside signal to drop this address, or you drop the mailbox.

