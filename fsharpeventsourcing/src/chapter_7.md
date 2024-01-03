# EventStore (database) 
An event-store, stores and retrieves events and snapshots related to each single context.

An example of a storage implementation in Postgres: 
[DbStorage.fs](https://github.com/tonyx/Sharpino/blob/main/Sharpino.Lib/PgEventStore.fs)

# EventStoreDb:

The alternative storage is the EventStoreBridge, which is a bridge to the EventStore database.
It is still experimental.
[EventStoreStorage.fs](https://github.com/tonyx/Sharpino/blob/main/Sharpino.Lib/EventStoreStorage.fs)


