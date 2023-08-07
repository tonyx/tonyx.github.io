# Introduction

[Sharpino](https://github.com/tonyx/Micro_ES_FSharp_Lib) is a simple F# event sourcing framework.
It is a project for study and for experimenting.
At the moment (2023-08-06) it supports _in memory_ and _postgres_ storages.
There is also an experimental support for eventstoredb.

Here I am giving a quick overview to how it works, how the sample application works (from models to application service layer and without any user interface) and my proposal about the issues of refactoring aggregates (migrating to a different configuration of aggregates).


## The sample application

The sample application is a simple todo list manager where each todo contains reference to Tags and Categories.
The aggregates manage those categories, todos and tag models.
Particularly the "version 1" of the saple application has an Aggregate managing Todo and Categories, and another aggregate managing Tags.
I will show a strategy for handling the following aggregate refactoring problem: how to migrate from an aggregate configuration to another one.
 
Particularly there in the "version 2" of the application there are three different aggregates: one for the Todo model, one for the Categories model, and another one from the Tags model.

The challenge is that you can write parametric tests that can be executed against the current and the next aggregate configuration, also by testing the migration function.

