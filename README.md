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

The list of events is saved using event store (usually using scalable solutions like [Apache Kafka](https://kafka.apache.org/) or [EventStore](https://geteventstore.com/)) which is also used as a message bus.

![eventual consistency](https://cloud.githubusercontent.com/assets/1868852/25046843/f53e6c90-2133-11e7-93de-76f0b13b8c8b.png)

In this diagram, there are three isolated instances, each updating its own database as new events are dispatched from the event store.

It is possible that these instances are not in sync at all times, but we can still query them.

It's irrelevant what language or database we use in any of these instances.

New instances can easily be added by replaying all events in the event store and then subscribing to new events.

### Match Making Example
![Match Making](https://cloud.githubusercontent.com/assets/1868852/25047642/4333ffa6-2138-11e7-96be-5c7d2fda9585.png)

The flow of data:
1. **Users** aggregate receives a command from "the outside" (over ws, http, rpc). The command is then transformed into an event and saved in the event store.
1. **Process Manager** listens to all events in the store and based on these events, issues a command.
1. In case the user had requested a matchmaking, "find match" command is sent to **SQS** handler.
1. **MM** instance receives a command and tries to find possible match
1. If the match is found, *POSSIBLE_MATCH_FOUND* event is dispatched.
1. **Process Manager** reacts to *POSSIBLE_MATCH_FOUND* event and checks if userIdA and userIdB are indeed (or still) a good match.
1. If match is validated, "start game" command is sent to **Users** aggregate
1. If match is no longer valid (Elo rank was changed in the meantime, user went offline), "Find Match" command is again initiated

Described architecture is not a complete implementation of "Event sourcing" pattern.
The problem is, there is a (small) probability of inconsistency between **dynamoDB** (main database) and event store (in case messege failed to be stored).

One way this can be solved is by using events from the event store for creating the main database:

![matchmaking](https://cloud.githubusercontent.com/assets/1868852/25057349/50711352-216f-11e7-9b2b-069e9b568286.png)

Unfortunately, there could be a lot of work to be done in existing arhitectures, because every db create/update/delete would need to be changed in event source pattern.

## Using a portion of users
Another (optional) way of improving matchmaking performance is to use a sample of users.

If, for example, there are 1 000 000 users currently online and ten instances are in charge of finding a match,
each instance could work with a part of the data set.
Instead of querying over 1 million users, 100 000 users is a user-base which is large enough for finding a good match.

Users per instance can also be categorised by geo-location parameter.
