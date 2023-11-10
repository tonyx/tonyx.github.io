# Introduction

[Sharpino](https://github.com/tonyx/Micro_ES_FSharp_Lib) is a little F# event sourcing framework.
Even though the example provided are closer to the concept of 
At the moment (2023-08-06) it supports _in memory_, _Postgres_ and _EventStoreDb as storage for events.

The framework is based on the concept of _clusters_ and _command_ around clusters.
A cluster is a set of collections of entities. A command is a request to a cluster to perform an action involving one or more entities.

This doc is quick overview of how Sharpino works by means of its sample application: a Todo list manager with tags and categories. Tags and categories work as a way to classify todos. We will see the difference between having a single cluster managing all the entities and having separate clusters for different entities.

I'll focus on: entities, cluster, events, commands, service application layer, repository and storages.
I'll also mention the problem of undoing a command and how to handle cluster refactoring (moving entities from one cluster to another and at the same time testing all those configurations).

## The sample application

As already mentioned the sample application is a todo list manager. Each todo contains references to Tags and Categories.
I'll show how we can handle two versions of the same application that use different cluster configurations. This will help to talk about the cluster refactoring.

The "version 1" has a cluster managing Todo and Categories, and another cluster managing Tags.

The "version 2" of the application consists of three different clusters: one for the Todo model, one for the Categories model, and another one for the Tags model.

The challenge about being able to write parametric tests that can be executed against the current and the next cluster configuration at the same time, having the migration between the former and the latter configuration.
That means that you may avoid caring about the cluster configuration when prototyping the application. Creating cluster will means no more than partitioning the set of the collections of entities. 

A prototype of an application may start with a single cluster managing all the entities. Then, when the application grows, you may want to split the cluster in two or more clusters. This is what I call "cluster refactoring".

