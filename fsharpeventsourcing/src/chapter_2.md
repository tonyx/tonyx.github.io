# Aggregates

An aggregate is a class that handles one or more models. The state of an aggregate can be rebuild by using snapshots and by processing stored events.

An aggregate must define the following static members:

- __Zero__: an instance of the aggregate in its intitial state. 
- __StorageName__ and  __Version__: this combination uniquely identifies the aggreagate  to make sure the storage will store snapshots and events in dedicated tables.
- __LockObject__: the repository uses them to lock the aggregate while storing related events ensuring consistency. an application layer may use them explicitly to ensure inter-aggregate integrity (invariant conditions involving models in different aggregates).
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
        static member LockObj =
            LockObject.Instance.LokObject
        static member SnapshotsInterval =
            15
```

Some member of the aggregate has the role of "changing" its state. They are functions returning a new instance of the aggregate with the new state in a Result type. Those members will be assicated to specific events (see next section).

Example:
```FSharp
    member this.AddTodo (t: Todo) =
        let checkCategoryExists (c: Guid ) =
            this.categories.GetCategories() 
            |> List.exists (fun x -> x.Id = c) 
            |> boolToResult (sprintf "A category with id '%A' does not exist" c)

        ResultCE.result
            {
                let! categoriesMustExist = t.CategoryIds |> catchErrors checkCategoryExists // FOCUS HERE
                let! todos = this.todos.AddTodo t
                return 
                    {
                        this with
                            todos = todos
                    }
            }
```


Source: [TodosAggregate.fs](https://github.com/tonyx/Micro_ES_FSharp_Lib/blob/main/Micro_ES_FSharp_Lib.Sample/aggregates/Todos/Aggregate.fs)
