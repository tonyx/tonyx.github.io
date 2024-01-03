# Command Handler
The Command Handler has the responsibility of:
- getting the state
- trying to run commands passed, and eventually storing the related events.
- making periodic snapshots, according to the SnapshotsInterval.

There are two different command handler implementations:


[CommandHandler.fs](https://github.com/tonyx/Sharpino/blob/main/Sharpino.Lib/CommandHandler.fs)


There is also an experimental repository based on a publish/subscribe storage model (Eventstoredb).
See _lightrepository_

[LightCommandHandler.fs](https://github.com/tonyx/Sharpino/blob/main/Sharpino.Lib/LightCommandHandler.fs)

Note: from version 1.4.7 the commandHandler can use a Kafka broker to retrieve the current state.
That means that the commandHandler can be used in a distributed environment, where the state is stored in a Kafka topic.
Moreover, it can anyway rely on the EventStoreDb to rebuild the current state in case of failure of the Kafka broker (for example when events are out of sync).