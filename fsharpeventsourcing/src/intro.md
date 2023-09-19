# Introduction

[Sharpino](https://github.com/tonyx/Micro_ES_FSharp_Lib) is a little F# event sourcing framework.
At the moment (2023-08-06) it supports _in memory_ and _Postgres_ storage.
There is also experimental support for _EventstoreDB_.

This doc is quick overview of how Sharpino works by means of its sample application.
I'll talk about: entities, aggregates, events, commands, service application layer, repository and storages.
I'll also mention the problem of undoing a command and how to handle aggregate refactoring.

## The sample application

The sample application is a todo list manager. Each todo contains references to Tags and Categories.
I'll show two versions of the same application that use different aggregate configurations. This will help to talk about the aggregate refactoring.

The "version 1" has an Aggregate managing Todo and Categories, and another aggregate managing Tags.

The "version 2" of the application consists of three different aggregates: one for the Todo model, one for the Categories model, and another one for the Tags model.

The challenge about being able to write parametric tests that can be executed against the current and the next aggregate configuration at the same time, having the migration between the former and the latter configuration.
That means that you may avoid caring about the aggregate configuration when prototyping the application. The concept of aggregate will __not__ be about having an aggregate root. Creating aggreagatre will means no more than partitioning the set of the collections of entities. (each aggregate take care of a partition of the set of all the entities)

