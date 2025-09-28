# Testing

A simple way to test multiple configuration is base on passing the different instances of the application (for instance one is using the in memory event store and another one is using the postgres event store). 
A simple extension to Expecto is provided introducint the "multipleTestCase" function. Here it is used to test also a migration between aggregate versions

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








