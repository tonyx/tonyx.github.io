# Refactoring strategy

By cluster refactoring, I mean when we just move models (collections of entities) among contexts.

Here I am showing a strategy for refactoring in terms of:  
- moving the model's ownership between contexts, 
- introducing new contexts 
- upgrading old contexts.
- dropping contexts.

The problem arises because it looks overcomplicated d to make upfront decisions about contexts. Consider that an application may start, for simplicity, with a single context and then, at a later stage, we may want to split it into multiple contexts.

Refactoring leaves the application service layer behavior unchanged.

The steps that may be followed are:
- defining new contexts and eventually creating upgraded versions of current contexts
- moving collection of entities from contexts to others.
- creating an upgraded version of the application service layer using the new versions
- applying the equivalent tests of the previous service layer to the new one.

About this latest point, a parametric testing strategy is also possible.

Here is an example of a list of tuples of multiple application version configurations with migration functions.
 
```FSharp
let allVersions =
    [
        (applicationPostgresStorage,        applicationPostgresStorage,       fun () -> () |> Result.Ok)
        (applicationShadowPostgresStorage,  applicationShadowPostgresStorage, fun () -> () |> Result.Ok)
        (applicationPostgresStorage,        applicationShadowPostgresStorage, applicationPostgresStorage._migrator.Value)

        (applicationMemoryStorage,          applicationMemoryStorage,         fun () -> () |> Result.Ok)
        (applicationShadowMemoryStorage,    applicationShadowMemoryStorage,   fun () -> () |> Result.Ok)
        (applicationMemoryStorage,          applicationShadowMemoryStorage,   applicationMemoryStorage._migrator.Value)
    ]
```

There are specific attributes to distinguish current and "upgrading" versions of elements of the application. 

A migration function is needed to extract data from the current version and store it in the upgraded version.

Code here: [MultiVersionsTests.fs](https://github.com/tonyx/Sharpino/blob/main/Sharpino.Sample.Test/MultiVersionsTests.fs)

