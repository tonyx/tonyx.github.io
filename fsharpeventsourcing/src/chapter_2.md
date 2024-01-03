# Context

A context is just a group of collections of entities.
Roughly speaking, a context is a group of entities that are related to each. We may use the adjective "bounded" to describe a context.
Context will be associated with streams of events.
We can get the state of any cluster by processing the stored events using the _evolve_ function.  _events_ are closely related to members of the cluster that end up in Update/Delete/Remove of some entity (even though there is no actual change because we are using immutable data structures).

Static members that are mandatory for any cluster of entities are:
- __Zero__: the initial state (no events yet).
- __StorageName__ and  __Version__: this combination uniquely identifies the cluster and lets the storage know in which stream to store events and snapshots. Whatever will be the storage (memory, Postgres, EventstoreDb, etc.) the cluster will be stored in a stream named as the concatenation of the _StorageName_ and the _Version_ (i.e. "_todo_01")
- __Lock__: a lock object is used to protect the cluster from concurrent access (will be used by the Command handler if the PessimisticLock is set to true in config).

- __SnapshotsInterval__: the number of events that can be stored after a snapshot before creating a new snapshot (i.e. the number of events between snapshots)

The Command handler, by the runCommand function, applies a command, then stores the related events and returns the EventStore (database) Ids of those stored events and the KafkaDeliveryResult (if Kafka broker is enabled).

Example of a cluster of entities handling the todos and the categories:
```FSharp
    type TodosCluster =
        {
            todos: Todos
            categories: Categories
        }
        static member Zero =
            {
                todos = Todos.Zero
                categories = Categories.Zero
            }
        static member StorageName =
            "_todo"
        static member Version =
            "_01"
        static member SnapshotsInterval =
            15
```

In the following example, the TodosContext can check the validity of the categories referenced by any todo before adding it (to preserve the invariant rule that you can add only todo with valid category ID references).
It uses the "result" computational expression included in the FsToolkit.ErrorHandling library which supports the [_railway-oriented programming_](https://fsharpforfunandprofit.com/rop/) pattern of handling errors.

Example:
```FSharp
    member this.AddTodo (t: Todo) =
        let checkCategoryExists (c: Guid ) =
            this.categories.GetCategories() 
            |> List.exists (fun x -> x.Id = c) 
            |> boolToResult (sprintf "A category with id '%A' does not exist" c)

        result
            {
                let! categoriesMustExist = t.CategoryIds |> catchErrors checkCategoryExists
                let! todos = this.todos.AddTodo t
                return 
                    {
                        this with
                            todos = todos
                    }
            }
```

Todo: [Context.fs](https://github.com/tonyx/Sharpino/blob/main/Sharpino.Sample/Domain/Todos/Context.fs)
