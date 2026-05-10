# The Domain Model & Pure Aggregates

In event-sourced systems, the **Aggregate** is the primary boundary of consistency. It receives commands, applies business rules, and emits events.

## Ditching "Contexts"

In earlier versions of Sharpino, there was a concept known as a "Context" — essentially an event-sourced object that did not have a specific ID, assuming a single global instance.

**Contexts are now deprecated.** 

To streamline the architecture, Sharpino now advocates using **Pure Aggregates** for everything. Every event-sourced entity must have a proper `Id`. 

If your domain requires a "singleton" or single-instance object (like a global configuration or a master catalog), you achieve this by simply using an Aggregate with a *constant Id*. You then ensure at application startup that the instance exists (creating it automatically if it doesn't). This allows all optimizations, caching, and features to be developed exclusively for Aggregates without needing to support a separate "Context" construct.

## Combating Primitive Obsession

Aggregates require an ID, often represented by a standard `Guid`. However, relying heavily on raw Guids leads to "Primitive Obsession," where the compiler cannot distinguish between a `Book` Guid and an `Author` Guid.

Taking inspiration from the `blazorBookLibrary`, Sharpino encourages defining specific domain-relevant Value Objects for IDs. By wrapping the Guid in a strongly typed construct (e.g., `BookId`, `AuthorId`), you leverage the F# type system to guarantee compile-time safety and prevent accidental cross-pollination of identifiers across different aggregate types.
