# Match Making
There are two main ideas on how to improve speed and scalability of a long running process like "matchmaking":
- Eventual consistency
- Using a portion of users

## Eventual Consistency
In CRUD style architecture, where a single database is used for almost all computations there are a number of issues: database locking, hard to scale, lost of history.

Most importantly, CRUD-style approach is favouring application consistency,
and there is [no way](https://en.wikipedia.org/wiki/CAP_theorem) a system can be fully scalable and fully consistent at the same time.

So, to improve scalability, it would be wiser to stop favouring consistency where it's not needed.

If the system is designed in a way that every new event is saved in some kind of database as a simple list, that can be used as a source of information for creating multiple databases.

This is a pattern known as "Event sourcing".

If you are not familliar with it, this is a great introductory video on the subject:
[YOW! Nights March 2016 Martin Fowler - Event Sourcing](https://www.youtube.com/watch?v=aweV9FLTZkU)

The list of events is saved using event store (usually using scalable solutions like [Apache Kafka](https://kafka.apache.org/) or [EventStore](https://geteventstore.com/)) which is also used as a message bus.

![eventual consistency](https://cloud.githubusercontent.com/assets/1868852/25046843/f53e6c90-2133-11e7-93de-76f0b13b8c8b.png)

In this diagram, there are three isolated instances, each updating its own database as new events are dispatched from the event store.

It is possible that these instances are not in sync at all times, but we can still query them.

It's irrelevant what language or database we use in any of these instances.

New instances can easily be added by replaying all events in the event store and then subscribing to new events.
The same is true for instances whose database needs to be rebuilt.

### Match Making Example
![matchmaking](https://cloud.githubusercontent.com/assets/1868852/25308309/65c82806-27b1-11e7-98ed-b0580154db14.png)

The flow of data:
1. **Users** aggregate receives a command from "the outside" (over ws, http, rpc). The command is then transformed into an event and saved in the event store.
1. **Process Manager** listens to all events in the store and based on these events, issues a command.
1. In case the user had requested a matchmaking, "find match" command is sent to **SQS** handler.
1. **MM** instance receives a command and tries to find possible match
1. If the match is found, *POSSIBLE_MATCH_FOUND* event is dispatched.
1. **Process Manager** reacts to *POSSIBLE_MATCH_FOUND* and sens a "start game" command to **Users** aggregate
1. **Users** aggregate validates a command and checks if userIdA and userIdB are indeed (or still) a good match.
1. If command is accepted, "GAME_STARTED" event is saved
1. If match is declined (Elo rank was changed in the meantime, user went offline), "Find Match" command is again initiated

## Using a portion of users
Another (optional) way of improving matchmaking performance is to use a sample of users.

If, for example, there are 1 000 000 users currently online and ten instances are in charge of finding a match,
each instance could work with a part of the data set.
Instead of querying over 1 million users, 100 000 users is a user-base which is large enough for finding a good match.

Users per instance can also be categorised by geo-location parameter.

Because of this fact, MM instances could use the fastest database possible: in-memory (RAM).
