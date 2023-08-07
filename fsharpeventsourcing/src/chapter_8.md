# Refactoring strategy

By aggregate refactoring I mean when we have the same models, the same application behavior and we just move models from one aggregate to another.

Here I am showing a strategy for refactoring aggregates in terms of:  
- moving the model's ownership between aggregates, 
- introducing new aggregates 
- upgrading old aggregates.
- dropping aggregates.

The problem arises because it looks overcomplicated d to make upfront decisions about aggregates.

It looks to me more convenient to have a few aggregates, for instance in a development and prototyping stage because this will simplify testing, prototyping, and building the application service layer.
Consider the extreme when we have everything in a single aggregate. It would mean that the application service layer will be able to handle all the models with few lines of code (just building single command for the repository)

However, at a later stage, proper refactoring is probably needed by moving models to different aggregates or creating new aggregates for performance reasons.

Refactoring aggregates in that sense means leaving the application service layer behavior unchanged.

The steps that may be followed are:
- defining new aggregates and eventually creating upgraded versions of current aggregates
- moving models ownership from old aggregates to new aggregates (or to updated versions of the same aggregates which is the same)
- creating an upgraded version of the application service layer using the new set of aggregates
- applying the equivalent tests of the previous service layer to the new one.

About this latest point, a parametric testing strategy is also possible.

Here is an example of a list of tuples of multiple application version configurations with migration functions.
 
```FSharp
let allVersions =
    [
        (AppVersions.applicationPostgresStorage,        AppVersions.applicationPostgresStorage,       fun () -> () |> Result.Ok)
        (AppVersions.applicationShadowPostgresStorage,  AppVersions.applicationShadowPostgresStorage, fun () -> () |> Result.Ok)
        (AppVersions.applicationPostgresStorage,        AppVersions.applicationShadowPostgresStorage, AppVersions.applicationPostgresStorage._migrator.Value)

        (AppVersions.applicationMemoryStorage,          AppVersions.applicationMemoryStorage,         fun () -> () |> Result.Ok)
        (AppVersions.applicationShadowMemoryStorage,    AppVersions.applicationShadowMemoryStorage,   fun () -> () |> Result.Ok)
        (AppVersions.applicationMemoryStorage,          AppVersions.applicationShadowMemoryStorage,   AppVersions.applicationMemoryStorage._migrator.Value)
    ]
```

There are specific attributes to distinguish current and "upgrading" versions of elements of the application. 

A migration function is needed to move data from old aggregates to new aggregates.

Code here: [MultiVersionsTests.fs](https://github.com/tonyx/Micro_ES_FSharp_Lib/blob/main/Sharpino.Sample.Test/MultiVersionsTests.fs)

