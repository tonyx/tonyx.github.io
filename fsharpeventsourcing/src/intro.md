# Introduction

[Sharpino](https://github.com/tonyx/Micro_ES_FSharp_Lib) is a simple F# event sourcing framework.
It is a project for study and for experimenting.
At the moment (2023-08-06) it supports _in memory_ and _postgres_ storages.
There is also an experimental support for eventstoredb.

Here I am giving a quick overview to how it works, how the sample application works (from models to application service layer and without any user interface) and how to handle aggregate refactoring.




## The sample application

The sample application is a simple todo list manager where each todo contains reference to Tags and Categories.
There are two version of the same application that use different aggregate configurations.

The "version 1" has an Aggregate managing Todo and Categories, and another aggregate managing Tags.

The "version 2" of the application consists in three different aggregates: one for the Todo model, one for the Categories model, and another one for the Tags model.

The challenge is that you can write parametric tests that can be executed against the current and the next aggregate configuration, also by testing the migration function.

