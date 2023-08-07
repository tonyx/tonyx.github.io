# Storage
A storage stores and retrieves events and snapshots related to each single aggregate.

In a strict way storage ensures transactionality particularly in special cases like when adding events related to multiple aggregates.

In a general sense, there is the possibility of supporting storages that lack transactionality between multiple streams,  and that is the case of the _EventStoreBridge_ (or, in the future, Kafa etc..)

Here is the abstract definition of members required for storage:

```FSharp
    abstract member Reset: version -> Name -> unit
    abstract member TryGetLastSnapshot: version -> Name -> Option<int * int * Json>
    abstract member TryGetLastEventId: version -> Name -> Option<int>
    abstract member TryGetLastSnapshotEventId: version -> Name -> Option<int>
    abstract member TryGetLastSnapshotId: version -> Name -> Option<int>
    abstract member TryGetEvent: version -> int -> Name -> Option<StorageEvent>
    abstract member SetSnapshot: version -> int * Json -> Name -> Result<unit, string>
    abstract member AddEvents: version -> List<Json> -> Name -> Result<unit, string>
    abstract member MultiAddEvents:  List<List<Json> * version * Name>  -> Result<unit, string>
    abstract member GetEventsAfterId: version -> int -> Name -> List<int * string >
```

Reset must be used only for development and testing, and cannot be used in production. See Conf.fs.

An example of a storage implementation in Postgres: [DbStorage.fs](https://github.com/tonyx/Sharpino/blob/main/Sharpino.Lib/DbStorage.fs)

# EventStoreBridge:

The alternative storage is the EventStoreBridge, which is a bridge to the EventStore database.
It is still experimental.
[EventStoreBridge.fs](https://github.com/tonyx/Sharpino/blob/main/Sharpino.Lib.EventStore/EventStoreBridge.cs)


