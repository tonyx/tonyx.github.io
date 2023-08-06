# Introduction

[Sharpino](https://github.com/tonyx/Micro_ES_FSharp_Lib) is a simple F# event sourcing library for .NET Core.
It is a project for study and for experimenting.
At the moment (2023-08-06) it supports in memory and postgres storage for events and snapshots.
I started implementing also Eventstoredb.

This document gives a quick overview of the framework, the sample application (only api for any client, no user intrface) and some hints about a refactoring and migration strategy.

## The sample application
By sample application I mean the business logic without any user interface.
The sample application is a simple todo list manager.
Such todo contains reference to Tags and Categories.
Tags, categories and todos are models that are managed by aggregates.
I will show a solution of the aggregate refactoring problem: how to migrate from a specific aggregate configuration to another one. Particularly there is the version 1 of the application where we have an aggregate for Todos and Categories, and another aggregate for Tags. The refactoring consist on the migration in a configuraion where we have three different aggregates: Todos, Categories and Tags.

