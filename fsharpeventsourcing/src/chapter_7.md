# Storage
A storage stores and retrieves events and snapshots related to each single aggregate.

In a strict way storage ensures transactionality particularly in special cases like when adding events related to multiple aggregates.

In a general sense, there is the possibility of supporting storages that lack transactionality between multiple streams,  and that is the case of the _EventStoreBridge_ (or, in the future, Kafa etc..)

Here is the abstract definition of members required for storage:

```FSharp
        abstract member Reset: version -> Name -> unit
        abstract member TryGetLastSnapshot: version -> Name -> Option<int * int * 'A>
        abstract member TryGetLastEventId: version -> Name -> Option<int>
        abstract member TryGetLastSnapshotEventId: version -> Name -> Option<int>
        abstract member TryGetLastSnapshotId: version -> Name -> Option<int>
        abstract member TryGetEvent: version -> int -> Name -> Option<StorageEvent<obj>>
        abstract member SetSnapshot: version -> int * 'A -> Name -> Result<unit, string>
        abstract member AddEvents: version -> List<'E> -> Name -> Result<unit, string>
        abstract member MultiAddEvents:  List<List<obj> * version * Name>  -> Result<unit, string>
        abstract member GetEventsAfterId: version -> int -> Name -> Result< List< int * 'E >, string >
        abstract member GetEventsInATimeInterval: version -> Name -> DateTime -> DateTime -> List<int * 'E >
```

I created also a "lightweight" version of the storage interface assuming that there are storage types where you will not need all the full storage interface.

Reset must be used only for development and testing, and cannot be used in production. See Conf.fs.

An example of a storage implementation in Postgres: [DbStorage.fs](https://github.com/tonyx/Sharpino/blob/main/Sharpino.Lib/DbStorage.fs)

# EventStoreBridge:

The alternative storage is the EventStoreBridge, which is a bridge to the EventStore database.
It is still experimental.
[EventStoreBridge.fs](https://github.com/tonyx/Sharpino/blob/main/Sharpino.Lib/EventStoreStorage.fs).


