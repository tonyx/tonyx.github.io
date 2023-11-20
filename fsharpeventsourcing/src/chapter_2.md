# Clusters

A cluster is just a group of collections of entities that form a transactional boundary.
Clusters will be associated with streams of events.
You can get the current state of any cluster by processing the stored events using the _evolve_ function.  _events_ are closely related to members of the cluster that end up in Update/Delete/Remove of some entity (even though there is no actual change because we are using immutable data structures).

Static members that are mandatory for any cluster of entities are:
- __Zero__: the initial state (no events yet).
- __StorageName__ and  __Version__: this combination uniquely identifies the cluster and lets the storage know in which stream to store events and snapshots. Whatever will be the storage (memory, Postgres, EventstoreDb, etc.) the cluster will be stored in a stream named as the concatenation of the _StorageName_ and the _Version_ (i.e. "_todo_01")
- __Lock__: a lock object is used to protect the cluster from concurrent access.
The Command handler uses Locks. An application layer may use locks when involves multiple streams of events.

- __SnapshotsInterval__: the number of events that can be stored after a snapshot before creating a new snapshot (i.e. the number of events between snapshots)

The Command handler returns the updated state of any cluster by applying the events that are stored after the latest valid state (which is cached in memory).
The Command handler is also able to rebuild the state starting from the last stored snapshot.

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

Some instance members virtually change the state in the sense that they return a new instance in a different state or an error. It means that such members will _add/update/delete_ some of their entities. you need to associate those members with _events_ and _commands_ (see next section).
In the following example, the TodosCluster can check the validity of the categories referenced by any todo before adding it (to preserve the invariant rule that you can add only todo with valid category ID references).
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
Conclusions: In adding/removing/updating entities in a cluster we can protect any invariant conditions related to other entities that are under the same cluster without the need for any explicit transaction. We can protect other invariant conditions that may involve entities of other clusters anyway. This will be possible at the service application layer (see in the next sections) running commands involving multiple streams of events. This sometimes will involve the use of explicit transactions when supported by the storage (i.e. Postgres), and sometimes will involve the "undoer": a special command attached to a command to undo the changes made by the command itself (see the next sections), useful when the storage does not support multiple stream transactions.

Source code: [TodosCluster.fs](https://github.com/tonyx/Sharpino/blob/main/Sharpino.Sample/clusters/Todos/Cluster.fs)
