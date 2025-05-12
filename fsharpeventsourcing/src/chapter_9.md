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








