# Refactoring strategy

Here I am showing a strategy for refactoring aggregates in terms of:  
- moving the models ownership between aggregates, 
- introducing new aggregates 
- upgrading old aggregates.
- dropping aggregates.

Looks more convenient having few aggregate, in a development and prototyping stage.

This will simplify testing, prototyping, building the application service layer.

However, at a later stage, a proper refactoring is probably needed by moving models to different aggregates or creating new aggregates for performance reasons.

Refactoring aggregates in that sense means leaving the application service layer behavior unchanged.

The steps that may be followed are:
- defining new aggregates and eventually create upgraded version of current aggregates
- moving models ownership from old aggregates to new aggregates (or to updated versions of the same aggregates which is the same)
- creating an upgraded version of the application service layer using the new set of aggregates
- applyng the equivalent tests of the previous service layer to the new one.

About this latest point, a parametric testing strategy is also possible.

Here is anexample of a list of tuples of multiple application version configuration with migration function.
 
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

There are specific attributes to distinguish current and "upgrading" verion of elements of the application. 

A migration function is needed to move data from old aggregates to new aggregates.

Code here: [MultiVersionsTests.fs)[https://github.com/tonyx/Micro_ES_FSharp_Lib/blob/main/Sharpino.Sample.Test/MultiVersionsTests.fs]

