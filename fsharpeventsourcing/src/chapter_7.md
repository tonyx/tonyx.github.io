# Storage
A storage stores and retrieves events and snapshots related to each single cluster.

I created also a "lightweight" version of the storage interface assuming that there are storage types where you will not need all the full storage interface.

Reset must be used only for development and testing, and cannot be used in production. See Conf.fs.

An example of a storage implementation in Postgres: [DbStorage.fs](https://github.com/tonyx/Sharpino/blob/main/Sharpino.Lib/DbStorage.fs)

# EventStoreBridge:

The alternative storage is the EventStoreBridge, which is a bridge to the EventStore database.
It is still experimental.
[EventStoreBridge.fs](https://github.com/tonyx/Sharpino/blob/main/Sharpino.Lib/EventStoreStorage.fs).


