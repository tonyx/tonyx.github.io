# Clusters

You can define a cluster as a record with static and instance members. It handles one or more collections of entities. The crucial role of the cluster is that I can get the current state of any cluster by processing the stored events using the _evolve_ function. As we will see later, the _events_ are closely related to the members of the cluster that are meant to "virtually" change the state of the cluster (there is no actual change because we are using immutable data structures).

Given that the state of the cluster is a function of the related events processed and stored, it needs the following information associated that I defined as mandatory, static members:

- __Zero__: the initial state (no events yet).
- __StorageName__ and  __Version__: this combination uniquely identifies the cluster and lets the storage know in which stream to store events and snapshots. Whatever will be the storage (memory, Postgres, EventstoreDb, etc.) the cluster will be stored in a stream named as the concatenation of the _StorageName_ and the _Version_ (i.e. "_todo_01")

- __Lock__: 
An application layer can rely on locks to ensure inter-cluster integrity (invariant conditions involving models handled by separate clusters).

Wrap up: we may use no lock at all and just accept that unconsistent events may be stored because of concurrent processing, or we may use locks to ensure that commands are processed one at a time, or we may use mailboxprocessor to use single thread processing for any command.

- __SnapshotsInterval__: the number of the events that can be stored after a snapshot before creating a new snapshot (i.e. the number of events between snapshots)

In all my current implementations of storage  (_in memory_,  _Postgres_ and _EventstoreDb_)  the state of any cluster is rebuilt starting from the latest available snapshot and applying the events that are after the snapshot. The current state of any cluster may also be cached.

Example:
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

Some instance members virtually change the state in the sense that they return a new  instance in a different state or an error. It means that such members will _add/update/delete_  some of their entities. We will be able to associate those members with  _events_ (see next section).
In the following example, the TodosCluster manages the todos and the categories entities so it will be able to check the validity of the categories referenced by any todo before adding it (to preserve the rule that you can add only todo with category-Ids related to existing categories).
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
Wrap-up: Wile adding/removing/updating entities in a clustee we can protect any invariant conditions related to other entities that we have put under the same cluster without the need for any explicit transaction because at that level there is no awareness of the storage. Still, I will show that we can protect other invariant conditions that may involve entities of other clusters. This will be possible at the service application layer (see in the next sections) running commands involving multiple streams streams of events. This sometimes will involve the use of explicit transactions when supported by the storage (i.e. Postgres), and sometimes will involve the "undoer": a special command attached to a command to undo the changes made by the command itself (see the next sections), useful when the storage does not support multiple stream transactions.

Source code: [TodosCluster.fs](https://github.com/tonyx/Sharpino/blob/main/Sharpino.Sample/clusters/Todos/Cluster.fs)
