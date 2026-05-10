# Events and Commands

The heartbeat of Sharpino relies on two fundamental concepts: **Commands** (the intent to change state) and **Events** (the immutable record of that change).

## Events

Events represent facts that have already occurred. Because they form the historical ledger of the system, they must be immutable and safely serializable. When an Aggregate processes a Command, it yields a list of Events. These events are appended to the Event Store and then processed to derive the Aggregate's new state.

## Commands

Commands are functions that encapsulate business intent. They receive the current state of an Aggregate, along with any necessary parameters, validate the business rules, and return a `Result`. If successful, the Result contains the Events to be committed. Note that for aggregates we need to use `AggregateCommand` from Sharpino.Core instead of `Command`.

### The De-emphasis of the "Undoer"

Previously, Sharpino supported an "Undoer" mechanism for Commands—an automatic way to generate reverse actions to rollback a command. 

In modern implementations, the use of the "Undoer" is heavily de-emphasized. In practice, it is almost always set to `None`. 

Relying on generic "undo" logic often breaks domain semantics. Instead, compensating actions or state reversions should be modeled explicitly as their own domain Commands (e.g., instead of undoing a `PayOrder` command, issue an explicit `RefundOrder` command). This keeps the domain model clear, explicit, and intention-revealing.
