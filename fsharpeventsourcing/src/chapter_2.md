# Aggregates

An aggregate is an instance of a record that handles one or more models. We will be able to recreate the state of the aggregate by processing the stored events by a fundamental function called _evolve_. The definitions of such _events_ are closely related to some of the members of the aggregate. 

Given that the state of the aggregatre is a function of the related events present in the storage, an aggregate needs the following, mandatory, static members:

- __Zero__: the instance of the aggregate in its intitial state. 
It is needed to be able to provide the state of the aggregate when there is no event stored related to that aggregate.
- __StorageName__ and  __Version__: this combination uniquely identifies the aggragate and lets the storage know in which stream to store events and snapshots.

- - __LockObject__: (_warning: the lockobject concept is obsolete. It was meant to handle single thread processing but at the moment I am providing an actor model based mailboxprocessor to ensure single thread chain command->events->eventstoring, so you will skip this part_). Before introducing the _mailboxprocessor_ (i.e. an actor model based single thread message processor), the repository was supposed to use aggregate locks while processing command and storing related events ensuring consistency. An application layer was also supposed to use them explicitly to ensure inter-aggregate integrity (invariant conditions involving models handled by separate aggregates).
- __SnapshotsInterval__: the number of the events that can be stored after a snapshot before creating a new snapshot (i.e. the numer of events between snapshots)

Note that in the _in memory_ or _Postgres_ storage implementation the state of any aggregate is rebuilt starting from the last available snapshot and applying the events that are after the snapshot.
At the moment I have no specific strategy for hadling snapshots and interval between snapshots for the EventStoreDb. 

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

Some members of the aggregate virtually change the state of the aggregate in the sense that they they return the aggregate in a new state or an error. It atually means that such members will do add/upate/delete on their model (still in a functional/immutable way i.e. returning the model with changes). Those members will be associated to specific definition of _events_ (see next section).
In the following example the aggregate of the todos manages also the categories model and so it can check the validity of the categories referenced before adding a todo (you can add only todo with categoryIds related to existing categories).
It uses the "result" computational expression included in the FsToolkit.ErrorHandling library supporting the possibility to handle errors in a _railway oriented programming_ style.

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
Recap: invariant conditions related to models residing in the same aggregate can be preserved without the need of explicit transaction, because at that level there is no awareness of the storage. Invariant conditions involving models residing in different aggregates could be preserved as well, but only by the application layer level (see the next sections).

Source: [TodosAggregate.fs](https://github.com/tonyx/Sharpino/blob/main/Sharpino.Sample/aggregates/Todos/Aggregate.fs)
