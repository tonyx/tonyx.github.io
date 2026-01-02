# EventStore (database) 
An eventstore stores and retrieves events and snapshots related to each single context.
There are plenty of members to store and retrieve events and snapshots in different ways. Particularly some of them may use explicitly the Task return type and cancellation token. The standard behavior is still the use of a globally configured timeout.

Some (but not all) the commandHandler functions use the Task and optiona cancellation token as well. The goal is going for a full TaskResult based Api with optional cancellation token in the next major release.

The Postgres (json and binary) implementations use optimistic concurrency by checking the eventId before storing new events.

The inmemory implementation is useful for testing and prototyping.

An example of a storage implementation in Postgres: 
[PgEventStore.fs](https://github.com/tonyx/Sharpino/blob/main/Sharpino.Lib/PgEventStore.fs)



