# Introduction

[Sharpino](https://github.com/tonyx/Micro_ES_FSharp_Lib) is a little F# event sourcing framework.
_Contexts_:  Objects that can be event sourced. Only one instance of a context can exist at a time.
_Aggregates_: Object that can be event sourced. Many instances of an aggregate can exist at a time.

Contexts and aggregates need to define members that do Add/Remove/Update operations functionally and by using Result type to handle errors.

Contexts and aggregates need event types associated with Add/Remove/Update operations.
Event types need associated commands
Commands and events must be Discriminated Union types.

This document is a guide to the Sharpino library. It is a work in progress.
Focus on the following topics:
entities, contexts, events, commands, service application layer, Event Store.

## The sample application 1.

A Todo list manager. Each todo contains references to Tags and Categories.
I'll show how we can handle two versions of the same application that use different cluster configurations. This will help to talk about the cluster refactoring.

The "version 1" has a context managing Todo and Categories, and another cluster managing Tags.

The "version 2" of the application consists of three different clusters: one for the Todo model, one for the Categories model, and another one for the Tags model.

## Sample application 2. Booking system for seats in a stadium
Contexts represent rows. Some constraints are applied to the rows. The context is responsible for the seats. The seats are associated with the rows.

## Sample application 4. Booking system for seats in a stadium
The same as the application 2 but where rows are aggregates (so I can have any number of instances of seat rows). The application uses the SAFE stack based template.