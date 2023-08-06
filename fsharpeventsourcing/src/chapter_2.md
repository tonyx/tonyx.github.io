# Aggregates

An aggregate is a class that handles one or more models, preserving their invariant conditions. We will be able to access to the state of the aggregate by processing the stored events (aggregate state is function of events).

An aggregate must define the following static members:

- __Zero__: an instance of the aggregate in its intitial state. 
- __StorageName__ and  __Version__: this combination uniquely identifies the aggragate  and let the storage know in which stream to store events and snapshots.

- - __LockObject__: (_warning: the lockobject concept is obsolete. At the moment I am providing an actor model based mailboxprocessor to ensure single thread chain command->events->eventstoring, so you will skip this part_). the repository uses them to lock the aggregate while storing related events ensuring consistency. an application layer may use them explicitly to ensure inter-aggregate integrity (invariant conditions involving models in different aggregates).
- __SnapshotsInterval__: the number of the events that can be stored after a snapshot before creating a new snapshot (i.e. the numer of events between snapshots)
- __Zero__: the aggregate's initial state, when no events happened yet.

Example:
```FSharp
    type TodosAggregate =
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

Some members of the aggregate  virtually changes its state (i.e. they return the aggregate in a new state or an error).  Those members will be assiociated to specific events data structure (see next section).
In the following example the aggregate of the todos manages also the categories model and so it provides check related to categories before adding a todo (only todo with existing category ids can be added).
It uses a computational expression included in the FsToolkit.ErrorHandling library to manage errors.

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


Source: [TodosAggregate.fs](https://github.com/tonyx/Sharpino/blob/main/Sharpino.Sample/aggregates/Todos/Aggregate.fs)
