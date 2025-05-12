# Command Handler
The Command Handler has the responsibility of:
- getting the state
- trying to run commands passed, and eventually storing the related events.
- making periodic snapshots, according to the SnapshotsInterval.

There are two different command handler implementations:

[CommandHandler.fs](https://github.com/tonyx/Sharpino/blob/main/Sharpino.Lib/CommandHandler.fs)

