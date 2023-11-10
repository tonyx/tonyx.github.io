# Refactoring strategy

By cluster refactoring I mean when we just move models (collection of entities) among clusters.

Here I am showing a strategy for refactoring in terms of:  
- moving the model's ownership between cluster, 
- introducing new clusters 
- upgrading old clusters.
- dropping clusters.

The problem arises because it looks overcomplicated d to make upfront decisions about clusters.

It looks to me more convenient to have a few clusters, for instance in a development and prototyping stage because this will simplify testing, prototyping, and building the application service layer.
Consider the extreme when we have everything in a single cluster. It would mean that the application service layer will be able to handle all the models with few lines of code (just building single command for the repository)

However, at a later stage, proper refactoring is probably needed for performance reasons.

Refactoring leaves the application service layer behavior unchanged.

The steps that may be followed are:
- defining new clusters and eventually creating upgraded versions of current clusters
- moving entities ownership
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

Code here: [MultiVersionsTests.fs](https://github.com/tonyx/Micro_ES_FSharp_Lib/blob/main/Sharpino.Sample.Test/MultiVersionsTests.fs)

