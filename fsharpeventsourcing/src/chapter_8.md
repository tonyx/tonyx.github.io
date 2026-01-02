# Refactoring Aggregates via upcasting

If you need to change an aggregate (or context) for new requirements, you need to refactor the aggregate so that it can handle the old and the new formats.

An upcasting technique is based on being able to create an aggregate that mimics the old one and an upcast function to convert from the old one to the new one.

It is convenient to do a bulk upcast of all the existing aggregates by making a snapshot of all of them to be able to drop the old aggregate definition from the code.
The new snapshots will use the new aggregate format, so the definition of the old aggregate will be unnecessary after the bulk upcast and resnapshot.

To do an upcast, you can create a shadow aggregate that mimics the old one in the same module of the current one, provided that the module is defined as rec (recursive).

The mechanism is based on handling the failure of the Deserialization by using as fallback the Deserialization using the old aggregate and then the upcast function to the new one.

Note: it may depend on the serialization library and its configuration as it is not always true that you can deserialize the old aggregate changing its target type with a different name than the original one. 

The provided examples are all using FSPickler configured in a way that it allows the upcast from the old aggregate with a different name to the new one.


Quick example of an upcast from an old aggregate to a new one:

The current version of the Course aggregate is:
```fsharp
type Curse =
    {
        Id: Guid
        Name: string
        Students: List<Guid>
        MaxNumberOfStudents: int
    }
```

We want to add a new field _Teachers_ to the aggregate so it will become:
```fsharp
type Course =
    {
        Id: Guid
        Name: string
        Students: List<Guid>
        MaxNumberOfStudents: int
        Teachers: List<Guid>
    }
```

We want to be able to read the old version of the aggregate and upcast it to the new one. So we redefine a type _Course001_ that mimics the old version of the aggregate:
```fsharp
type Course001 =
    {
        Id: Guid
        Name: string
        Students: List<Guid>
        MaxNumberOfStudents: int
    }
```

We instrument the course001 with deserialization and upcast function

```fsharp
type Course001 =
    {
        Id: Guid
        Name: string
        Students: List<Guid>
        MaxNumberOfStudents: int
        Teachers: List<Guid>
    }

    with
        static member Deserialize x =
            jsonPSerializer.Deserialize<Curse001> x
        member this.Upcast(): Course =
            {   Name = this.Name
                Id = this.Id
                Students = this.Students
                MaxNumberOfStudents = this.MaxNumberOfStudents
                Teachers = []
            }

Finallly we make sure that the current deserialize function of the Course aggregate can handle both the old and the new format:
```fsharp 
    static member Deserialize(x: string): Result<Course, string> =
        let firstTry = jsonPSerializer.Deserialize<Course> x
        match firstTry with
        | Ok x -> x |> Ok
        | Error e ->
            let secondTry = jsonPSerializer.Deserialize<Curse001> x
            match secondTry with
            | Ok x -> x.Upcast() |> Ok
            | Error e1 ->
                Error (e + " " + e1)
```                    

Examples using FSharp.SystemTextJson are also privided. 
It's important to test any library to be sure that the "mimic old type" deserialization and upcast works as in this example.


