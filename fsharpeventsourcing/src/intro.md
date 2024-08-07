# Introduction

[Sharpino](https://github.com/tonyx/Micro_ES_FSharp_Lib) is a little F# event sourcing framework.
_Contexts_:  Objects that can be event sourced. There is no specific id for a context so we just assume that a single instance and a single zero (initial state) instance exist for any _Context_.
_Aggregates_: Objects that can be event sourced. Many instances of an aggregate can exist at a time so we need an Id for each instance represented by a Guid. The initial state for any aggregate will end up in an initial snapshot in the event store.

Contexts and aggregates need to define members that do Add/Remove/Update operations functionally and by using Result type to handle errors. We tipically use Railway oriented programming is used to handle errors in any Add/Remove/Update operation (by the FsTookit.ErrorHandling library).

Contexts and aggregates need to specify event types associated with Add/Remove/Update operations. 
Event types need associated commands
Commands and events must be Discriminated Union types implementing the _Command_ and the _Event_ interfaces respectively.

This document is a guide to the Sharpino library. It is a work in progress.
Focus on the following topics:
Contexts, events, commands, service application layer, Event Store.

## The sample application 1.

This sample is convoluted and represent unlikely situations like having complex relations and interactions between different aggregates/contexts. I would suggest to 
skip this part. I'd say the interesting proposal in this example is about the refactoring technique: how to move from a specific distribution of concerns between contexts/aggregate to another distribution.
How to test the migration of the context/aggregate.
The idea is to create a new version of context/aggregate and at the same time build a migration function.

Each todo contains references to Tags and Categories.
I'll show how we can handle two versions of the same application that use different cluster configurations. This will help to talk about the context refactoring.

The "version 1" has a context managing Todo and Categories, and another context managing Tags.

The "version 2" of the application consists of three different contexts: one for the Todo model, one for the Categories model, and another one for the Tags model.

## Sample application 2. Booking system for seats in a stadium
Contexts represent rows. Some constraints are applied to the rows. The context is responsible for the seats. The seats are associated with the rows.

## Sample application 4. Booking system for seats in a stadium
The same as application 2 but where rows are aggregates (so I can have any number of instances of seat rows). The application uses the SAFE based stack based template son can publish the system as a rest service using the Fable Remoting library.