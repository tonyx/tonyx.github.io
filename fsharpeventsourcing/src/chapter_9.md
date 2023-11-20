# Testing

As I mentioned in the previous chapter we may have different versions of the same application based on different clusters.

We may want to test all of them and also we may want to test the migration function from one version to another in those tests.
A structure of a parametric test that considers various possible combinations of application versions and migration functions is the following:

```FSharp

        multipleTestCase "test something " versions <| fun (ap, apUpgd, migrator)  ->
            let _ = ap._reset()

            // test something on ap, which is the "initial version" 
            // of the application
            .....

            // then migrate from "ap" version to apUpgd version 
            // using its migrator function

            let migrated = migrator()
            Expect.isOk migrated "should be ok"

            // test something on apUpgd, which is the "upgraded version" of the application

```

In some cases, you just want to run your tests fast and you don't want to test everything against all the possible combinations of application versions.

Then you may just skip some of the versions set up in the "version" triples __(initversion, endversion, migrator)__ by commenting out temporarily the ones you don't want to spend time to test at the moment:

```FSharp
let allVersions =
    [

        // (AppVersions.currentPostgresApp,        AppVersions.currentPostgresApp,     fun () -> () |> Result.Ok)
        // (AppVersions.upgradedPostgresApp,       AppVersions.upgradedPostgresApp,    fun () -> () |> Result.Ok)
        // (AppVersions.currentPostgresApp,        AppVersions.upgradedPostgresApp,    AppVersions.currentPostgresApp._migrator.Value)

        (AppVersions.currentMemoryApp,          AppVersions.currentMemoryApp,       fun () -> () |> Result.Ok)
        // (AppVersions.upgradedMemoryApp,         AppVersions.upgradedMemoryApp,      fun () -> () |> Result.Ok)
        // (AppVersions.currentMemoryApp,          AppVersions.upgradedMemoryApp,      AppVersions.currentMemoryApp._migrator.Value)

        // (AppVersions.evSApp,                    AppVersions.evSApp,                 fun () -> () |> Result.Ok)
    ]

```

The above code enables only the tests of the current version of the app that uses the in-memory storage.








