# Command Handler
The Command Handler has the responsibility of running the commands which means doing the following steps:
- get the state/s of the aggregate/s invoved on the command.
- try to run commands on that states, 
- try to store the resulting events,
- store the new state in the cache
- send the events to the event broker/message bus (if any), using "fire and forget" 
- make periodic snapshots, according to the SnapshotsInterval.
Those steps are enveloped by a result computation expression expressing explicitly the "happy path" and implicitly the "error path" using monadic
operators.

Note: the failure in sending events or in making snapshot is not relevant. Only the fact that the event is stored in the event store is relevant.

To address any consistency issue from the side of a message listener (i.e., a listener that reads events from a message broker and updates its own state accordingly),
the message listener can/should implement a "fallback" policy so thy they can be rehydrated by a call to the event store to restore the correct state of the aggregate.


[CommandHandler.fs](https://github.com/tonyx/Sharpino/blob/main/Sharpino.Lib/CommandHandler.fs)

