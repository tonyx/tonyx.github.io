# Introduction

[Sharpino](https://github.com/tonyx/Micro_ES_FSharp_Lib) is a little F# event sourcing framework.
At the moment (2023-08-06) it supports _in memory_, _Postgres_ and _EventStoreDb as storage for events.
The framework is based on the concept of _aggregate_ and _command_ around aggregates.
An aggregate is a set of collections of entities. A command is a request to an aggregate to perform an action involving one or more entities.

This doc is quick overview of how Sharpino works by means of its sample application: a Todo list manager with tags and categories. Tags and categories work as a way to classify todos. We will see the difference between having a single aggregate managing all the entities and having separate aggregates for different entities.

I'll focus on: entities, aggregates, events, commands, service application layer, repository and storages.
I'll also mention the problem of undoing a command and how to handle aggregate refactoring (moving entities from one aggregate to another and at the same time testing all those configurations).

## The sample application

As already mentioned the sample application is a todo list manager. Each todo contains references to Tags and Categories.
I'll show how we can handle two versions of the same application that use different aggregate configurations. This will help to talk about the aggregate refactoring.

The "version 1" has an Aggregate managing Todo and Categories, and another aggregate managing Tags.

The "version 2" of the application consists of three different aggregates: one for the Todo model, one for the Categories model, and another one for the Tags model.

The challenge about being able to write parametric tests that can be executed against the current and the next aggregate configuration at the same time, having the migration between the former and the latter configuration.
That means that you may avoid caring about the aggregate configuration when prototyping the application. The concept of aggregate will __not__ be about having an aggregate root (which is the classic definition). Creating aggreagate will means no more than partitioning the set of the collections of entities. (each aggregate take care of a partition of the set of all the entities)

A prototype of an application may start with a single aggregate managing all the entities. Then, when the application grows, you may want to split the aggregate in two or more aggregates. This is what I call "aggregate refactoring".

