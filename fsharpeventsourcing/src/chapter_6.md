# Command Handler
The Command Handler has the responsibility of:
- getting the state
- trying to run commands passed, and eventually storing the related events.
- making periodic snapshots, according to the SnapshotsInterval.

There are two different command handler implementations:


Code in [CommandHandler.fs](https://github.com/tonyx/Sharpino/blob/main/Sharpino.Lib/CommandHandler.fs)
[Core.fs](https://github.com/tonyx/Micro_ES_FSharp_Lib/blob/main/Sharpino.Lib/Core.fs)


There is also an experimental repository based on a publish/subscribe storage model (Eventstoredb).
See _lightrepository_

[LightCommandHandler.fs](https://github.com/tonyx/Sharpino/blob/main/Sharpino.Lib/LightCommandHandler.fs)

