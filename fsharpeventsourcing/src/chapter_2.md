# Aggregates

An aggregate is a class that handles one or more models in a consistent way, taking care of integrities between the models. The state of an aggregate can be rebuild by using snapshots and by processing stored events.

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

Some members of the aggregate  virtually changes it state (i.e. return the aggregate in a new state or error).  Those members will be assicated to specific events (see next section).
In the following example the aggregate of the todos manage also the categories models and so it provides checkings on the categories models when adding a todo.
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
