# Introduction

[Sharpino](https://github.com/tonyx/Micro_ES_FSharp_Lib) is a simple F# event sourcing framework.
It is a project for study and for experimenting.
At the moment (2023-08-06) it supports _in memory_ and _Postgres_ storage.
There is also experimental support for _EventstoreDB_.

Here I am giving a quick overview of how it works, how the sample application works (from the entities to the application service layer and without any user interface) and how to handle aggregate refactoring.


## The sample application

The sample application is a todo list manager. Each todo contains references to Tags and Categories.
There are two versions of the same application that use different aggregate configurations.

The "version 1" has an Aggregate managing Todo and Categories, and another aggregate managing Tags.

The "version 2" of the application consists of three different aggregates: one for the Todo model, one for the Categories model, and another one for the Tags model.

The challenge is that you can write parametric tests that can be executed against the current and the next aggregate configuration, also by testing the migration between the former and the latter.

