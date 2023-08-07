# Aggregates

An aggregate is a class that contains and handle one or more models. We will be able to access to the state of the aggregate by processing the stored events where the definitions of such events are closely related to members of the aggregate. 

Given that the state of the aggregatre is a function of the related events present in the storage, an aggregate must define the following static members:

- __Zero__: an instance of the aggregate in its intitial state. 
It is needed to initialize the aggregate when no events are present in the storage.
- __StorageName__ and  __Version__: this combination uniquely identifies the aggragate and let the storage know in which stream to store events and snapshots.

- - __LockObject__: (_warning: the lockobject concept is obsolete. It was meant to handle single thread processing but at the moment I am providing an actor model based mailboxprocessor to ensure single thread chain command->events->eventstoring, so you will skip this part_). Before introducing the mailboxprocessor, the repository was supposed to use aggregate lock while processing command and storing related events ensuring consistency. An application layer was also supposed to use them explicitly to ensure inter-aggregate integrity (invariant conditions involving models placed in different aggregates).
- __SnapshotsInterval__: the number of the events that can be stored after a snapshot before creating a new snapshot (i.e. the numer of events between snapshots)

Note that in the _in memory_ or _Postgres_ storage implementation the state of any aggregate is rebuilt starting from the last available snapshot and applying the events that are after the snapshot.

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

Some members of the aggregate that are meant to do action on it changing (virtually) its state (i.e. they return the aggregate in a new state or an error). It atually means that such members will virtually change some models (add/upate/delete). Those members will be associated to specific events data structure (see next section).
In the following example the aggregate of the todos manages also the categories model and so it provides check related to categories before adding a todo (you can add only todo with categoryIds related to existing categories).
It uses a computational expression included in the FsToolkit.ErrorHandling library to manage errors in a _railway oriented programming_ style.

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
Recap: invariant conditions related to models residing in the same aggregate can be preserved without the need of explicit transaction, because at that level there is no awareness of the storage. Invariant conditions involving models residing in different aggregates must be preserved at the application layer level (see the next sections).

Source: [TodosAggregate.fs](https://github.com/tonyx/Sharpino/blob/main/Sharpino.Sample/aggregates/Todos/Aggregate.fs)
