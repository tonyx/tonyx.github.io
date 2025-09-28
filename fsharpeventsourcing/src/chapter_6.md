# Command Handler
The Command Handler has the responsibility of running the commands which means doing the following steps:
- get the state/s of the aggregate/s invoved on the command.
- try to run commands on that states, 
- try to store the resulting events,
- send the events to the event broker,
- making periodic snapshots, according to the SnapshotsInterval.
Those steps are enveloped by a result computation expression expressing explicitly the "happy path" and implicitly the "error path" using monading
operators.
Note: The only real success occurs after the event store is stored.
The delivery of the messages does not affect the result of running the command. 
Any message listener can implement a "fallback" policy so thy they can be rehydrated by a call to the event store to restore the correct state of the aggregate.


[CommandHandler.fs](https://github.com/tonyx/Sharpino/blob/main/Sharpino.Lib/CommandHandler.fs)

