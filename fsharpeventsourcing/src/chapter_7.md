# Storage
A storage stores and retrieves events and snapshots about any aggregate.

Storage must ensure transactionality particularly in special cases like when adding events related to multiple aggregates.

Here the abstract definition of members required for a storage:

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

Reset must be used only for development and test, and cannot be used in production. See Conf.fs.

An example of a storage implementation in postgres: [DbStorage.fs](https://github.com/tonyx/Sharpino/blob/main/Sharpino.Lib/DbStorage.fs)

# EventStoreBridge:

The alternative storage is the EventStoreBridge, which is a bridge to the EventStore database.
It is still experimental.
[EventStoreBridge.fs](https://github.com/tonyx/Sharpino/blob/main/Sharpino.Lib.EventStore/EventStoreBridge.cs)


