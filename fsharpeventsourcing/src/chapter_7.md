# Storage
A storage stores and retrieves events and snapshots related to each single cluster.

In a strict way storage ensures transactionality particularly in special cases like when adding events related to multiple clusters.

In a general sense, there is the possibility of supporting storages that lack transactionality between multiple streams,  and that is the case of the _EventStoreBridge_ (or, in the future, Kafa etc..)

Here is the abstract definition of members required for storage:

```FSharp
    type IStorage =
        abstract member Reset: Version -> Name -> unit
        abstract member TryGetLastSnapshot: Version -> Name -> Option<SnapId * EventId * Json>
        abstract member TryGetLastEventId: Version -> Name -> Option<EventId>
        // toto: the following two can be unified
        abstract member TryGetLastSnapshotEventId: Version -> Name -> Option<EventId>
        abstract member TryGetLastSnapshotId: Version -> Name -> Option<EventId * SnapshotId>
        abstract member TryGetSnapshotById: Version -> Name -> int ->Option<EventId * Json>
        abstract member TryGetEvent: Version -> int -> Name -> Option<StorageEventJson>
        abstract member SetSnapshot: Version -> int * Json -> Name -> Result<unit, string>
        abstract member AddEvents: Version -> Name -> List<Json> -> Result<List<int>, string>
        abstract member MultiAddEvents:  List<List<Json> * Version * Name>  -> Result<List<List<int>>, string>
        abstract member GetEventsAfterId: Version -> int -> Name -> Result< List< EventId * Json >, string >
        abstract member GetEventsInATimeInterval: Version -> Name -> DateTime -> DateTime -> List<EventId * Json >

```

I created also a "lightweight" version of the storage interface assuming that there are storage types where you will not need all the full storage interface.

Reset must be used only for development and testing, and cannot be used in production. See Conf.fs.

An example of a storage implementation in Postgres: [DbStorage.fs](https://github.com/tonyx/Sharpino/blob/main/Sharpino.Lib/DbStorage.fs)

# EventStoreBridge:

The alternative storage is the EventStoreBridge, which is a bridge to the EventStore database.
It is still experimental.
[EventStoreBridge.fs](https://github.com/tonyx/Sharpino/blob/main/Sharpino.Lib/EventStoreStorage.fs).


