# Introduction

[Sharpino](https://github.com/tonyx/Micro_ES_FSharp_Lib) is a simple F# event sourcing library. NET Core.
It is a project for study and for experimenting.
At the moment (2023-08-06) it supports _in memory_ and _postgres_ storages.
There is also an experimental support for eventstoredb.

Here I am giving a quick overview to how it works, the sample application (from models to application service layer and without any user interface) and my proposal about the issues of refactoring aggregates (migrating to a different configuration of aggregates).


## The sample application

The sample application is a simple todo list manager where each todo contains reference to Tags and Categories.
Tags, categories and todos are models that are managed by aggregates.
I will show a strategy for handling the following aggregate refactoring problem:  how to migrate from an aggregate configuration to another one. Particularly there is the version 1 of the application where we have an aggregate for Todos and Categories, and another aggregate for Tags. The refactoring consist on the migration in a configuraion where we have three different aggregates: Todos, Categories and Tags.
The challenge is that you can write parametric tests that can be executed against any aggregate configuration.

